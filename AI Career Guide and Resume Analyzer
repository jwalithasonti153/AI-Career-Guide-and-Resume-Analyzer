"""Flask application for the AI Career Guide and Resume Analyzer."""

from pathlib import Path

from flask import Flask, jsonify, render_template, request
from werkzeug.utils import secure_filename

from services.ai_service import AIService, AIServiceError


ALLOWED_RESUME_EXTENSIONS = {"pdf", "doc", "docx"}
MAX_RESUME_BYTES = 5 * 1024 * 1024


def create_app() -> Flask:
    app = Flask(__name__)
    app.config["MAX_CONTENT_LENGTH"] = MAX_RESUME_BYTES + 64 * 1024
    ai_service = AIService()

    @app.get("/")
    def index():
        return render_template("index.html")

    @app.post("/api/career-guide")
    def career_guide():
        payload = request.get_json(silent=True) or {}
        name = str(payload.get("name", "")).strip()
        skills = str(payload.get("skills", "")).strip()
        target_role = str(payload.get("target_role", "")).strip()
        errors = {}
        if not name:
            errors["name"] = "Please enter your full name."
        if not skills:
            errors["skills"] = "Add at least one skill."
        if not target_role:
            errors["target_role"] = "Choose or enter a target role."
        if len(name) > 100 or len(skills) > 2000 or len(target_role) > 120:
            errors["form"] = "One or more fields exceed the allowed length."
        if errors:
            return jsonify({"error": "Please check your details.", "fields": errors}), 400
        try:
            return jsonify(ai_service.generate_guide(name, skills, target_role))
        except AIServiceError as error:
            return jsonify({"error": str(error)}), 502

    @app.post("/api/resume-analyzer")
    def resume_analyzer():
        target_role = str(request.form.get("target_role", "")).strip()
        resume = request.files.get("resume")
        if resume is None or not resume.filename:
            return jsonify({"error": "Choose a PDF, DOC, or DOCX resume."}), 400
        if not target_role:
            return jsonify({"error": "Enter a target role for the analysis."}), 400
        if len(target_role) > 120:
            return jsonify({"error": "The target role is too long."}), 400
        filename = secure_filename(resume.filename)
        extension = Path(filename).suffix.lower().lstrip(".")
        if extension not in ALLOWED_RESUME_EXTENSIONS:
            return jsonify({"error": "Only .pdf, .doc, and .docx files are supported."}), 400
        file_bytes = resume.read(MAX_RESUME_BYTES + 1)
        if len(file_bytes) > MAX_RESUME_BYTES:
            return jsonify({"error": "Resume files must be 5 MB or smaller."}), 413
        try:
            resume_text = ai_service.extract_resume_text(file_bytes, extension)
            if len(resume_text.strip()) < 80:
                return jsonify({"error": "The resume did not contain enough readable text to analyze."}), 422
            return jsonify(ai_service.analyze_resume(resume_text, target_role))
        except AIServiceError as error:
            return jsonify({"error": str(error)}), 422

    @app.errorhandler(413)
    def request_too_large(_error):
        return jsonify({"error": "The submitted file is too large. Maximum size is 5 MB."}), 413

    return app


app = create_app()


if __name__ == "__main__":
    app.run(debug=True)
