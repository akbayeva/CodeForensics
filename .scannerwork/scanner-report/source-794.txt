import os
import shutil
import zipfile
from flask import Flask, render_template, request, redirect, url_for, jsonify

app = Flask(__name__)
UPLOAD_FOLDER = 'uploads'
SCAN_FOLDER = 'scan_results'
SONAR_FOLDER = 'sonarqube_reports'

os.makedirs(UPLOAD_FOLDER, exist_ok=True)
os.makedirs(SCAN_FOLDER, exist_ok=True)
os.makedirs(SONAR_FOLDER, exist_ok=True)

# Главная страница
@app.route('/')
def index():
    return render_template('index.html')

# Загрузка файла и анализ
@app.route('/upload', methods=['POST'])
def upload():
    if 'file' not in request.files:
        return "No file part"
    file = request.files['file']
    if file.filename == '':
        return "No selected file"
    if file:
        file_path = os.path.join(UPLOAD_FOLDER, file.filename)
        file.save(file_path)

        # Распаковываем файл
        extract_path = os.path.join(UPLOAD_FOLDER, 'extracted')
        with zipfile.ZipFile(file_path, 'r') as zip_ref:
            zip_ref.extractall(extract_path)

        # Запускаем SonarScanner и получаем отчет
        analysis_result = run_sonarqube_analysis(extract_path, file.filename)

        # Перенаправляем на страницу SonarQube с результатами
        sonar_url = f"http://localhost:9000/dashboard?id=python_test"
        return redirect(sonar_url)

# Запуск SonarScanner
def run_sonarqube_analysis(source_folder, filename):
    """Запускает SonarScanner для анализа кода"""
    output_folder = os.path.join(SONAR_FOLDER, filename.replace('.zip', ''))
    os.makedirs(output_folder, exist_ok=True)

    # Используем существующий конфигурационный файл
    properties_path = os.path.join(source_folder, 'sonar-project.properties')

    # Проверяем, если файл конфигурации не существует, копируем его в рабочую директорию
    if not os.path.exists(properties_path):
        default_properties_path = '/Users/aigerimakbayeva/PycharmProjects/pythonProject4/sonar-project.properties'
        shutil.copy(default_properties_path, properties_path)

    # Запускаем SonarScanner
    os.system(f"sonar-scanner -Dproject.settings={properties_path}")

    # Возвращаем информацию об отчете
    return {"status": "Analysis completed", "project": filename}

if __name__ == "__main__":
    app.run(debug=True, port=5007)
