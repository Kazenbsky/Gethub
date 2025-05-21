# Gethub
Mp3
# app.py
from flask import Flask, request, render_template, redirect, url_for
import os
import zipfile
import subprocess
from werkzeug.utils import secure_filename

UPLOAD_FOLDER = 'uploads'
EXTRACT_FOLDER = 'extracted'
ALLOWED_EXTENSIONS = {'zip'}

app = Flask(__name__)
app.config['UPLOAD_FOLDER'] = UPLOAD_FOLDER
app.config['EXTRACT_FOLDER'] = EXTRACT_FOLDER

os.makedirs(UPLOAD_FOLDER, exist_ok=True)
os.makedirs(EXTRACT_FOLDER, exist_ok=True)

def allowed_file(filename):
    return '.' in filename and filename.rsplit('.', 1)[1].lower() in ALLOWED_EXTENSIONS

def execute_python_file(filepath):
    try:
        result = subprocess.run(['python', filepath], capture_output=True, text=True, timeout=10)
        return result.stdout + result.stderr
    except Exception as e:
        return str(e)

@app.route('/', methods=['GET', 'POST'])
def upload_file():
    if request.method == 'POST':
        if 'file' not in request.files:
            return "Aucun fichier envoyé."
        file = request.files['file']
        if file.filename == '':
            return "Nom de fichier vide."
        if file and allowed_file(file.filename):
            filename = secure_filename(file.filename)
            filepath = os.path.join(app.config['UPLOAD_FOLDER'], filename)
            file.save(filepath)

            # Extraction du zip
            with zipfile.ZipFile(filepath, 'r') as zip_ref:
                extract_path = os.path.join(app.config['EXTRACT_FOLDER'], filename.replace('.zip', ''))
                os.makedirs(extract_path, exist_ok=True)
                zip_ref.extractall(extract_path)

            # Analyse et exécution
            file_infos = []
            for fname in os.listdir(extract_path):
                full_path = os.path.join(extract_path, fname)
                if fname.endswith('.py'):
                    output = execute_python_file(full_path)
                    file_infos.append({'name': fname, 'type': 'python', 'output': output})
                elif fname.endswith('.html'):
                    file_infos.append({'name': fname, 'type': 'html', 'path': full_path})
                elif fname.endswith('.mp3') or fname.endswith('.m4a'):
                    file_infos.append({'name': fname, 'type': 'audio', 'path': full_path})
                elif fname.endswith('.mp4'):
                    file_infos.append({'name': fname, 'type': 'video', 'path': full_path})
                else:
                    file_infos.append({'name': fname, 'type': 'other'})

            return render_template('result_exec_media.html', files=file_infos)

    return render_template('upload.html')

if __name__ == '__main__':
    app.run(debug=True)

    <!-- templates/upload.html -->
<!doctype html>
<html>
<head><title>Upload ZIP</title></head>
<body>
    <h1>Uploader un fichier ZIP</h1>
    <form method=post enctype=multipart/form-data>
        <input type=file name=file>
        <input type=submit value=Uploader>
    </form>
</body>
</html>

<!-- templates/result_exec_media.html -->
<!doctype html>
<html>
<head><title>Résultats</title></head>
<body>
    <h1>Résultats du traitement :</h1>
    <ul>
    {% for file in files %}
        <li>
            <strong>{{ file.name }}</strong>
            {% if file.type == 'python' %}
                <pre>{{ file.output }}</pre>
            {% elif file.type == 'html' %}
                ➤ <a href="{{ file.path }}" target="_blank">Ouvrir HTML</a>
            {% elif file.type == 'audio' %}
                ➤ <audio controls>
                    <source src="{{ file.path }}" type="audio/mpeg">
                    Votre navigateur ne supporte pas l'audio.
                </audio>
            {% elif file.type == 'video' %}
                ➤ <video width="320" height="240" controls>
                    <source src="{{ file.path }}" type="video/mp4">
                    Votre navigateur ne supporte pas la vidéo.
                </video>
            {% else %}
                ➤ (Fichier non exécutable)
            {% endif %}
        </li>
    {% endfor %}
    </ul>
    <a href="/">Retour</a>
</body>
</html>

# requirements.txt
Flask
Werkzeug
