# ====================
# DEPENDENCIES & IMPORTS
# ====================
from fastapi import FastAPI, Response
from fastapi.responses import HTMLResponse, StreamingResponse
from fastapi.middleware.cors import CORSMiddleware
import cv2
import pytesseract
from gtts import gTTS
import asyncio
from io import BytesIO

# ====================
# CONFIGURATION
# ====================
# Replace this with the actual URL of your ESP32-CAM's live stream.
video_url = 'http://<YOUR_ESP32_CAM_IP_ADDRESS>:81/stream'

# Set the path to the Tesseract executable (if it's not in your PATH).
# Example for Windows:
# pytesseract.pytesseract.tesseract_cmd = r'C:\Program Files\Tesseract-OCR\tesseract.exe'

# Global objects to manage video capture and application state
# The camera instance is created once at startup.
camera = cv2.VideoCapture(video_url)
last_extracted_text = ""

# ====================
# API SETUP
# ====================
# Create a FastAPI app instance
app = FastAPI(title="ESP32-CAM Fast OCR & TTS API")

# Configure CORS to allow requests from any origin
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# A simple HTML frontend to interact with the API.
FRONTEND_HTML = """
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>ESP32-CAM Fast OCR & TTS API</title>
    <style>
        body { font-family: 'Inter', sans-serif; background-color: #f0f4f8; color: #333; display: flex; justify-content: center; align-items: center; min-height: 100vh; margin: 0; }
        .container { background-color: white; padding: 2rem; border-radius: 12px; box-shadow: 0 4px 20px rgba(0,0,0,0.1); width: 90%; max-width: 600px; text-align: center; }
        h1 { color: #007bff; margin-bottom: 1.5rem; }
        p { margin-bottom: 1.5rem; color: #555; }
        .button-group { display: flex; justify-content: center; gap: 1rem; }
        button { padding: 12px 24px; border: none; border-radius: 8px; background-color: #007bff; color: white; font-size: 16px; cursor: pointer; transition: background-color 0.3s, transform 0.1s; }
        button:hover { background-color: #0056b3; }
        button:active { transform: scale(0.98); }
        .result-box { margin-top: 2rem; padding: 1.5rem; border: 2px solid #007bff; border-radius: 10px; background-color: #e6f2ff; text-align: left; }
        .result-box h2 { margin-top: 0; color: #0056b3; }
        #loading { margin-top: 1rem; color: #777; font-style: italic; display: none; }
        .error-message { color: #d9534f; margin-top: 1rem; }
        #audio-player { width: 100%; margin-top: 1rem; }
    </style>
    <script>
        // Store the state of the app
        let lastExtractedText = '';

        async function performOcr() {
            const resultBox = document.getElementById('result-box');
            const loadingIndicator = document.getElementById('loading');
            const audioPlayer = document.getElementById('audio-player');
            const extractedTextElement = document.getElementById('extracted-text');
            
            extractedTextElement.textContent = '';
            audioPlayer.style.display = 'none';
            loadingIndicator.style.display = 'block';

            try {
                // Step 1: Fetch and extract text from the OCR API
                const textResponse = await fetch('/get_text');
                if (!textResponse.ok) {
                    throw new Error(Text extraction failed: ${textResponse.statusText});
                }
                const textData = await textResponse.json();

                if (textData.extracted_text) {
                    lastExtractedText = textData.extracted_text;
                    extractedTextElement.textContent = lastExtractedText;

                    // Step 2: Now fetch the audio for the extracted text
                    await fetchAndPlayAudio(lastExtractedText);
                } else {
                    extractedTextElement.textContent = textData.message || 'No text detected.';
                }
            } catch (error) {
                console.error('Error:', error);
                extractedTextElement.textContent = Error: ${error.message};
            } finally {
                loadingIndicator.style.display = 'none';
            }
        }

        async function fetchAndPlayAudio(textToSpeak) {
            const audioPlayer = document.getElementById('audio-player');
            const url = '/get_audio';
            
            try {
                const response = await fetch(url, {
                    method: 'POST',
                    headers: {
                        'Content-Type': 'application/json'
                    },
                    body: JSON.stringify({ text: textToSpeak })
                });

                if (!response.ok) {
                    throw new Error(Audio generation failed: ${response.statusText});
                }

                const audioBlob = await response.blob();
                const audioUrl = URL.createObjectURL(audioBlob);

                audioPlayer.src = audioUrl;
                audioPlayer.style.display = 'block';
                audioPlayer.play();
            } catch (error) {
                console.error('Audio fetch error:', error);
                document.getElementById('extracted-text').textContent = Audio Error: ${error.message};
            }
        }
    </script>
</head>
<body>
    <div class="container">
        <h1>ESP32-CAM Fast OCR & TTS API</h1>
        <p>Click the button to capture a frame from your camera, perform OCR, and play the detected text as speech.</p>
        <div class="button-group">
            <button onclick="performOcr()">Capture & Process</button>
        </div>
        <div id="loading">Processing...</div>
        <div id="result-box" class="result-box">
            <h2>Result:</h2>
            <p id="extracted-text"></p>
            <audio id="audio-player" controls style="display: none;"></audio>
        </div>
    </div>
</body>
</html>
"""

@app.on_event("startup")
async def startup_event():
    """Event handler for when the application starts."""
    if not camera.isOpened():
        print(f"Failed to open video stream from URL: {video_url}. Is the ESP32-CAM on and reachable?")
        
@app.on_event("shutdown")
def shutdown_event():
    """Event handler for when the application shuts down."""
    camera.release()
    print("Camera stream released.")

@app.get("/", response_class=HTMLResponse)
async def read_root():
    """Serves the simple HTML frontend."""
    return FRONTEND_HTML

@app.get("/get_text")
async def get_text():
    """
    Captures a frame and returns the OCR-extracted text.
    Uses asyncio.to_thread to run the blocking OpenCV call in a separate thread,
    keeping the main event loop non-blocking.
    """
    global last_extracted_text

    def capture_and_process():
        """Blocking function to be run in a separate thread."""
        ret, frame = camera.read()
        if not ret:
            return None, "Could not read frame from stream."
        
        gray_frame = cv2.cvtColor(frame, cv2.COLOR_BGR2GRAY)
        text = pytesseract.image_to_string(gray_frame).strip()
        
        if text:
            return text, "Text extracted successfully."
        else:
            return None, "No text detected in the video frame."
    
    # Run the blocking function in a separate thread
    loop = asyncio.get_event_loop()
    text, message = await loop.run_in_executor(None, capture_and_process)

    if text:
        last_extracted_text = text  # Update the global state
        return {"extracted_text": text, "message": message}
    else:
        return {"extracted_text": None, "message": message}

@app.post("/get_audio")
async def get_audio(data: dict):
    """
    Generates and streams TTS audio for the given text.
    """
    text_to_speak = data.get("text", "No text to speak.")

    def generate_audio():
        """Blocking function to generate the audio file."""
        tts = gTTS(text=text_to_speak, lang='en')
        audio_stream = BytesIO()
        tts.write_to_fp(audio_stream)
        audio_stream.seek(0)
        return audio_stream
    
    # Run the blocking audio generation in a separate thread
    loop = asyncio.get_event_loop()
    audio_stream = await loop.run_in_executor(None, generate_audio)
    
    # Return the audio data as a streaming response
    return StreamingResponse(audio_stream, media_type='audio/mpeg')

# To run this file, you would use: uvicorn main:app --reload
