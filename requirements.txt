from flask import Flask, request, jsonify, send_from_directory
from moviepy.editor import *
from pydub import AudioSegment
import whisper, requests, os, uuid, subprocess, numpy as np

app = Flask(__name__, static_folder='static')

@app.route("/")
def index():
    return "✅ Railway video server is running!"

@app.route("/generate-video", methods=["POST"])
def generate_video():
    data = request.get_json()
    clips = data.get("clips", [])
    image_url = data.get("image_url", "")
    background_url = data.get("background_url", "")

    if not clips or not image_url or not background_url:
        return jsonify({"error": "Missing required inputs"}), 400

    try:
        os.makedirs("static", exist_ok=True)
        with open("scene.png", "wb") as f:
            f.write(requests.get(image_url).content)
        with open("background.mp3", "wb") as f:
            f.write(requests.get(background_url).content)

        audio_clips = []
        for i, clip in enumerate(clips):
            text = clip.get("voiceText", "")
            if not text.strip():
                continue
            silent = AudioSegment.silent(duration=2000 + 100 * len(text))
            path = f"voice_{i}.mp3"
            silent.export(path, format="mp3")
            audio_clips.append(AudioFileClip(path))

        scenes = []
        current_time = 0
        for audio in audio_clips:
            img = ImageClip("scene.png").set_duration(audio.duration).resize(height=720).set_position("center")
            scene = img.set_audio(audio).set_start(current_time)
            scenes.append(scene)
            current_time += audio.duration

        if not scenes:
            return jsonify({"error": "No valid scenes"}), 400

        base = concatenate_videoclips(scenes, method="compose")
        output_path = f"video_{uuid.uuid4().hex[:6]}.mp4"
        base.write_videofile(f"static/{output_path}", fps=24)

        return jsonify({ "video_url": f"/static/{output_path}" })

    except Exception as e:
        return jsonify({"error": str(e)}), 500

@app.route("/static/<path:filename>")
def static_files(filename):
    return send_from_directory("static", filename)

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=int(os.environ.get("PORT", 5000)))
