# Pywire
Interface that allows users to download YT videos.

## `index.html`:

```html
<!DOCTYPE html>
<html lang="en">
```
- Declares the document type as HTML5 and sets the language of the document to English.

```html
<head>
    <meta charset="UTF-8">
```
- Defines the character encoding for the HTML document, which is UTF-8, a common encoding format that supports most languages.

```html
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
```
- Ensures that the webpage is responsive by setting the viewport to scale properly on all device sizes.

```html
    <title>Pywire - Download YouTube Content</title>
```
- Sets the title of the web page that appears on the browser tab.

```html
    <link rel="icon" type="image/png" href="{{ url_for('static', filename='assets/favicon.png') }}">
    <link rel="stylesheet" href="{{ url_for('static', filename='style.css') }}">
</head>
```
- The first line links to the favicon image, which appears in the browser tab.
- The second line links to the external CSS file (`style.css`), which styles the page.

```html
<body>
    <div class="container">
```
- Begins the body section and creates a `div` with a `container` class to hold the main content.

```html
        <img src="{{ url_for('static', filename='assets/favicon.png') }}" alt="logo">
        <h1>Pywire</h1>
        <h3>Download YouTube Content</h3>
```
- Displays a logo (favicon) image at the top, followed by the main heading `Pywire` and a subheading `Download YouTube Content`.

```html
        <hr>
        <br>
```
- Inserts a horizontal line (`<hr>`) and a line break (`<br>`).

```html
        <form id="upload-form" action="/download" method="POST" enctype="multipart/form-data">
            <label for="url">YouTube URL:</label>
            <input type="text" id="url" name="url" required><br><br>
```
- Starts a form with a POST method that sends data to the `/download` route. The form includes a text input field where the user enters a YouTube URL.

```html
            <label for="download_type">Download Type:</label>
            <select id="download_type" name="download_type" required onchange="updateFormatOptions()">
                <option value="video">Video</option>
                <option value="audio">Audio</option>
            </select><br><br>
```
- Provides a dropdown (`select`) for the user to choose whether to download a video or audio. The `onchange` event triggers the `updateFormatOptions()` function to adjust format options dynamically.

```html
            <label for="format">Format:</label>
            <select id="format" name="format" required>
                <!-- Options will be dynamically populated based on download type -->
            </select><br><br>
```
- Adds another dropdown where the format options (video or audio) will be dynamically updated based on the download type selected.

```html
            <button type="submit">Download</button>
        </form>
```
- A button that submits the form to start the download process.

```html
        <!-- Loading Bar -->
        <div id="loading-bar-container">
            <div id="loading-bar"></div>
        </div>
        <div id="loading-text">Downloading, please wait...</div>
    </div>
```
- Adds a loading bar and text that will appear when the download is in progress.

### JavaScript Section:

```html
<script>
document.addEventListener('DOMContentLoaded', function() {
```
- Executes the enclosed code once the entire HTML document has loaded.

```javascript
    const videoFormats = JSON.parse('{{ video_formats|tojson|safe }}');
    const audioFormats = JSON.parse('{{ audio_formats|tojson|safe }}');
```
- These lines retrieve `video_formats` and `audio_formats` from the server, likely through a Jinja template, and parse them as JSON arrays.

```javascript
    function updateFormatOptions() {
        const downloadType = document.getElementById('download_type').value;
        const formatSelect = document.getElementById('format');
        formatSelect.innerHTML = '';  // Clear existing options
```
- This function updates the format dropdown based on whether the user selects video or audio as the download type.

```javascript
        const formats = downloadType === 'video' ? videoFormats : audioFormats;

        formats.forEach(format => {
            const option = document.createElement('option');
            option.value = format;
            option.textContent = format.toUpperCase();
            formatSelect.appendChild(option);
        });
    }
```
- Depending on the download type, it either fills the dropdown with video formats or audio formats.

```javascript
    updateFormatOptions();

    document.getElementById('download_type').addEventListener('change', updateFormatOptions);
```
- The format options are updated on page load and whenever the download type is changed by the user.

```javascript
    document.getElementById('upload-form').addEventListener('submit', function(event) {
        event.preventDefault();
```
- Prevents the default form submission to handle it via JavaScript.

```javascript
        document.getElementById('loading-bar-container').style.display = 'block';
        document.getElementById('loading-text').style.display = 'block';
```
- Shows the loading bar and text when the download starts.

```javascript
        const url = document.getElementById('url').value;
        const downloadType = document.getElementById('download_type').value;
        const format = document.getElementById('format').value;
```
- Retrieves the values entered by the user for URL, download type, and format.

```javascript
        const eventSource = new EventSource('/progress');
```
- Opens a connection to receive real-time progress updates via server-sent events (SSE) from the `/progress` endpoint.

```javascript
        eventSource.onmessage = function(event) {
            const data = JSON.parse(event.data);
            let progress = data.progress;
            let status = data.status;

            document.getElementById('loading-bar').style.width = progress + '%';
            document.getElementById('loading-text').textContent = `Processing... ${Math.round(progress)}%`;
```
- Updates the loading bar and text based on the progress received from the server.

```javascript
            if (status === "finished" || status === "error") {
                eventSource.close();
                document.getElementById('loading-text').textContent = "Process complete!";
            }
        };
```
- If the process finishes or an error occurs, the connection is closed, and the message is updated.

```javascript
        fetch('/download', {
            method: 'POST',
            headers: {
                'Content-Type': 'application/x-www-form-urlencoded',
            },
            body: `url=${encodeURIComponent(url)}&download_type=${encodeURIComponent(downloadType)}&format=${encodeURIComponent(format)}`
        }).then(response => response.blob()).then(blob => {
            const downloadUrl = URL.createObjectURL(blob);
            const a = document.createElement('a');
            a.href = downloadUrl;
            a.download = '';  // Optionally set the filename here
            document.body.appendChild(a);
            a.click();
            a.remove();
        }).catch(error => {
            console.error('Download failed:', error);
        });
    });
});
</script>
```
- Sends a POST request to the `/download` endpoint, initiates the download once the response is received, and allows the user to download the content.

```html
</body>
</html>
```
- Ends the body and HTML document.

---

## `app.py`:

```python
from flask import Flask, render_template, request, Response, stream_with_context
import yt_dlp
import os
import io
import tempfile
import zipfile
import urllib.parse
import logging
import re
import json
import time
```
- **Imports**: Imports necessary libraries:
  - `Flask`: Framework for handling web server logic.
  - `render_template`: To render HTML templates.
  - `request`: To handle incoming HTTP requests.
  - `Response`: To send custom responses from the server.
  - `stream_with_context`: To enable streaming responses.
  - `yt_dlp`: A Python package for downloading YouTube videos.
  - `os`, `io`, `tempfile`, `zipfile`: To handle file and directory operations.
  - `urllib.parse`: For URL encoding and decoding.
  - `logging`: To enable logging for debugging.
  - `re`: For working with regular expressions.
  - `json`: To handle JSON serialization/deserialization.
  - `time`: For handling delays and timestamps.

```python
app = Flask(__name__)
```
- **Flask App Initialization**: Initializes the Flask application.

```python
# Configure Flask and yt_dlp logging
logging.basicConfig(level=logging.DEBUG)
yt_dlp_logger = logging.getLogger('yt_dlp')
yt_dlp_logger.setLevel(logging.DEBUG)
console_handler = logging.StreamHandler()
console_handler.setLevel(logging.DEBUG)
yt_dlp_logger.addHandler(console_handler)
```
- **Logging Configuration**: Sets up logging to monitor both Flask and `yt_dlp`. This helps in debugging and outputs logs to the console.

```python
# Regular expression to match ANSI escape sequences
ansi_escape = re.compile(r'\x1B\[[0-?]*[ -/]*[@-~]')
```
- **ANSI Escape Code Regex**: Defines a regular expression to strip ANSI escape sequences from the log output, which are typically used for colored terminal output.

```python
# Global progress data
progress_data = {
    "status": "starting",
    "progress": 0,
    "current_item": 0,
    "total_items": 1,  # Initialize to 1 to avoid division by zero
}
```
- **Global Progress Data**: A dictionary to store the progress of downloads. It tracks the current status, progress percentage, the current item being downloaded, and the total number of items to download.

```python
def download_youtube_video(url, download_type, selected_format, temp_dir):
    global progress_data
    progress_data["current_item"] += 1
    progress_data["progress"] = 0  # Reset for each item
```
- **Download Function**: Defines the function to download a YouTube video. It increments the current item count and resets the progress for the new item.

```python
    def progress_hook(d):
        if d['status'] == 'downloading':
            progress_str = ansi_escape.sub('', d['_percent_str']).strip('%')
            downloaded_percentage = float(progress_str)
            total_items_progress = (progress_data['current_item'] - 1) * 100 + downloaded_percentage
            progress_data['progress'] = total_items_progress / progress_data['total_items']
            progress_data['status'] = 'downloading'
            logging.info(f"Overall progress: {progress_data['progress']}%")
        elif d['status'] == 'finished':
            logging.info("Download finished")
            progress_data['progress'] = (progress_data['current_item']) * 100 / progress_data['total_items']
            progress_data['status'] = 'processing'
```
- **Progress Hook**: A callback function to monitor the download progress, invoked by `yt_dlp`. It updates the `progress_data` with the percentage downloaded, and logs when the download finishes.

```python
    ydl_opts = {
        'format': 'bestvideo+bestaudio' if download_type == 'video' else 'bestaudio/best',
        'outtmpl': os.path.join(temp_dir, '%(title)s.%(ext)s'),
        'progress_hooks': [progress_hook],
    }
```
- **YouTube-DL Options**: Defines the options for `yt_dlp`:
  - Downloads the best available video and audio or the best audio based on the `download_type`.
  - Saves the file to the specified `temp_dir` with a filename based on the video title and extension.
  - Attaches the `progress_hook` to monitor download progress.

```python
    with yt_dlp.YoutubeDL(ydl_opts) as ydl:
        ydl.download([url])
```
- **Download Execution**: Uses `yt_dlp` to download the YouTube video using the specified options.

```python
@app.route('/')
def index():
    return render_template('index.html')
```
- **Route for Home Page**: Serves the `index.html` page when the root URL is accessed.

```python
@app.route('/download', methods=['POST'])
def download():
    url = request.form['url']
    download_type = request.form['download_type']
    selected_format = request.form['format']
```
- **Download Route**: Handles the form submission from the `index.html` page. It retrieves the YouTube URL, download type (video/audio), and format from the form.

```python
    if not re.match(r'^https?://(www\.)?(youtube\.com|youtu\.be)/', url):
        return 'Invalid URL', 400
```
- **URL Validation**: Checks if the provided URL is a valid YouTube URL. If not, returns a 400 Bad Request response.

```python
    try:
        with tempfile.TemporaryDirectory() as temp_dir:
            download_youtube_video(url, download_type, selected_format, temp_dir)
            filename = max([os.path.join(temp_dir, f) for f in os.listdir(temp_dir)], key=os.path.getctime)
            
            with open(filename, 'rb') as f:
                file_data = f.read()
                
            return Response(file_data, headers={
                'Content-Disposition': f'attachment; filename="{os.path.basename(filename)}"'
            })
    except Exception as e:
        logging.error(f"Error occurred: {str(e)}")
        return f"Error: {str(e)}", 500
```
- **Download Logic**: Creates a temporary directory to store the downloaded file. The `download_youtube_video` function is called to perform the actual download.
  - After the download completes, the most recently modified file is selected as the download result.
  - The file is read and returned as a downloadable response with a filename.

```python
@app.route('/progress')
def progress():
    def generate():
        total_items = 6  # Example: Total number of items to download/process
        for i in range(total_items):
            progress = (i / total_items) * 100
            yield f"data: {{\"progress\": {progress}, \"status\": \"downloading\"}}\n\n"
            time.sleep(1)  # Simulate processing time for each item
        
        yield f"data: {{\"progress\": 100, \"status\": \"finished\"}}\n\n"
    
    return Response(generate(), mimetype='text/event-stream')
```
- **Progress Endpoint**: Simulates a real-time progress update using Server-Sent Events (SSE). It streams progress updates (0% to 100%) back to the client, updating every second.

```python
# Run server
if __name__ == '__main__':
    app.run(debug=False)
```
- **Run Flask App**: Starts the Flask application if this script is run directly. `debug=False` ensures that the app runs in production mode.
