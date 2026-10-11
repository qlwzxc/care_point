from flask import Flask, render_template, request, redirect, url_for, flash, jsonify, Response
from prometheus_client import Counter, Gauge, generate_latest
from datetime import datetime, date, timedelta
import re
import sqlite3
import os
import time

app = Flask(__name__)
app.secret_key = os.environ.get("SECRET_KEY", "carepoint-demo-key")
DB_PATH = os.environ.get("DB_PATH", "carepoint.db")
started = time.time()
visits = []

REQUESTS = Counter("carepoint_http_requests_total", "Total HTTP requests")
APP_STATUS = Gauge("carepoint_app_up", "Application health status")
APP_STATUS.set(1)

DOCTORS = ["Терапевт", "Кардиолог", "Невролог", "Педиатр", "Дерматолог"]
STATUSES = ["Запланирована", "Состоялась", "Отменена"]
MAX_BOOKING_DAYS_AHEAD = 365


def db():
    connection = sqlite3.connect(DB_PATH)
    connection.row_factory = sqlite3.Row
    return connection


with db() as connection:
    connection.execute(
        """CREATE TABLE IF NOT EXISTS appointments (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            patient_name TEXT NOT NULL,
            phone TEXT NOT NULL,
            doctor TEXT NOT NULL,
            appointment_date TEXT NOT NULL,
            appointment_time TEXT NOT NULL,
            reason TEXT DEFAULT '',
            status TEXT DEFAULT 'Запланирована',
            created_at TEXT NOT NULL
        )"""
    )


def validate_appointment(name, phone, doctor, day_text, time_text, allow_past=False):
    """Validate appointment fields on the server; browser validation is not enough."""
    if not all([name, phone, doctor, day_text, time_text]):
        return "Заполни все обязательные поля."

    if len(name) < 2 or len(name) > 120:
        return "ФИО должно содержать от 2 до 120 символов."

    # Allow common phone formatting characters, but count the digits separately.
    if not re.fullmatch(r"[0-9+()\s-]+", phone):
        return "В телефоне допустимы только цифры, пробелы и символы + ( ) -."
    if phone.count("+") > 1 or ("+" in phone and not phone.lstrip().startswith("+")):
        return "Знак + можно поставить только в начале номера."
    digits = re.sub(r"\D", "", phone)
    if not 10 <= len(digits) <= 15:
        return "Номер телефона должен содержать от 10 до 15 цифр. Проверь номер."

    if doctor not in DOCTORS:
        return "Выбери врача из списка."

    try:
        appointment_day = datetime.strptime(day_text, "%Y-%m-%d").date()
    except (TypeError, ValueError):
        return "Укажи корректную дату приёма."

    if not allow_past and appointment_day < date.today():
        return "Нельзя записать на прошедшую дату."
    if appointment_day > date.today() + timedelta(days=MAX_BOOKING_DAYS_AHEAD):
        return "Запись доступна не более чем на год вперёд. Проверь дату."

    try:
        parsed_time = datetime.strptime(time_text, "%H:%M")
        if parsed_time.strftime("%H:%M") != time_text:
            raise ValueError
    except (TypeError, ValueError):
        return "Укажи корректное время приёма."

    return None


@app.before_request
def count_request():
    REQUESTS.inc()


@app.route("/")
def home():
    now = time.time()
    global visits
    visits = [visit for visit in visits if now - visit < 3600]
    visits.append(now)

    query = request.args.get("q", "").strip()
    status_filter = request.args.get("status", "")
    date_filter = request.args.get("date", "")
    sql = "SELECT * FROM appointments WHERE 1=1"
    args = []

    if query:
        sql += " AND (patient_name LIKE ? OR phone LIKE ? OR doctor LIKE ?)"
        like = f"%{query}%"
        args.extend([like, like, like])
    if status_filter in STATUSES:
        sql += " AND status=?"
        args.append(status_filter)
    if date_filter:
        try:
            datetime.strptime(date_filter, "%Y-%m-%d")
            sql += " AND appointment_date=?"
            args.append(date_filter)
        except ValueError:
            date_filter = ""

    sql += " ORDER BY appointment_date, appointment_time"
    with db() as connection:
        appointments = connection.execute(sql, args).fetchall()
        total = connection.execute("SELECT COUNT(*) FROM appointments").fetchone()[0]
        today = date.today().isoformat()
        today_count = connection.execute(
            "SELECT COUNT(*) FROM appointments WHERE appointment_date=? AND status!='Отменена'",
            (today,),
        ).fetchone()[0]
        cancelled = connection.execute(
            "SELECT COUNT(*) FROM appointments WHERE status='Отменена'"
        ).fetchone()[0]
        doctor_load = connection.execute(
            "SELECT doctor, COUNT(*) amount FROM appointments "
            "WHERE appointment_date>=? AND status!='Отменена' "
            "GROUP BY doctor ORDER BY amount DESC",
            (today,),
        ).fetchall()

    latest_day = (date.today() + timedelta(days=MAX_BOOKING_DAYS_AHEAD)).isoformat()
    return render_template(
        "index.html",
        appointments=appointments,
        doctors=DOCTORS,
        statuses=STATUSES,
        total=total,
        today_count=today_count,
        cancelled=cancelled,
        visits=len(visits),
        uptime=int(now - started),
        doctor_load=doctor_load,
        search=query,
        status_filter=status_filter,
        date_filter=date_filter,
        today=today,
        latest_day=latest_day,
    )


@app.post("/appointments/create")
def create():
    form = request.form
    name = form.get("patient_name", "").strip()
    phone = form.get("phone", "").strip()
    doctor = form.get("doctor", "")
    day_text = form.get("appointment_date", "")
    time_text = form.get("appointment_time", "")
    reason = form.get("reason", "").strip()

    error = validate_appointment(name, phone, doctor, day_text, time_text)
    if error:
        flash(error, "error")
        return redirect(url_for("home") + "#new")
    if len(reason) > 500:
        flash("Причина обращения не должна превышать 500 символов.", "error")
        return redirect(url_for("home") + "#new")

    with db() as connection:
        clash = connection.execute(
            "SELECT id FROM appointments WHERE doctor=? AND appointment_date=? "
            "AND appointment_time=? AND status!='Отменена'",
            (doctor, day_text, time_text),
        ).fetchone()
        if clash:
            flash("У этого врача уже есть запись на это время.", "error")
            return redirect(url_for("home") + "#new")
        connection.execute(
            "INSERT INTO appointments(patient_name, phone, doctor, appointment_date, "
            "appointment_time, reason, status, created_at) VALUES(?,?,?,?,?,?,?,?)",
            (
                name,
                phone,
                doctor,
                day_text,
                time_text,
                reason,
                STATUSES[0],
                datetime.now().isoformat(timespec="seconds"),
            ),
        )

    flash("Запись создана.", "success")
    return redirect(url_for("home"))


@app.post("/appointments/<int:aid>/edit")
def edit(aid):
    form = request.form
    name = form.get("patient_name", "").strip()
    phone = form.get("phone", "").strip()
    doctor = form.get("doctor", "")
    day_text = form.get("appointment_date", "")
    time_text = form.get("appointment_time", "")
    reason = form.get("reason", "").strip()
    status = form.get("status", "")

    error = validate_appointment(name, phone, doctor, day_text, time_text, allow_past=True)
    if error:
        flash(error, "error")
        return redirect(url_for("home"))
    if status not in STATUSES:
        flash("Выбери корректный статус записи.", "error")
        return redirect(url_for("home"))
    if len(reason) > 500:
        flash("Причина обращения не должна превышать 500 символов.", "error")
        return redirect(url_for("home"))

    with db() as connection:
        clash = connection.execute(
            "SELECT id FROM appointments WHERE doctor=? AND appointment_date=? "
            "AND appointment_time=? AND status!='Отменена' AND id!=?",
            (doctor, day_text, time_text, aid),
        ).fetchone()
        if clash:
            flash("У этого врача уже есть запись на это время.", "error")
            return redirect(url_for("home"))
        cursor = connection.execute(
            "UPDATE appointments SET patient_name=?, phone=?, doctor=?, appointment_date=?, "
            "appointment_time=?, reason=?, status=? WHERE id=?",
            (name, phone, doctor, day_text, time_text, reason, status, aid),
        )
        if cursor.rowcount == 0:
            flash("Запись не найдена.", "error")
            return redirect(url_for("home"))

    flash("Запись обновлена.", "success")
    return redirect(url_for("home"))


@app.post("/appointments/<int:aid>/delete")
def delete(aid):
    with db() as connection:
        cursor = connection.execute("DELETE FROM appointments WHERE id=?", (aid,))
    if cursor.rowcount == 0:
        flash("Запись не найдена.", "error")
    else:
        flash("Запись удалена.", "success")
    return redirect(url_for("home"))


@app.route("/health")
def health():
    try:
        with db() as connection:
            connection.execute("SELECT 1")
        return jsonify(
            status="healthy",
            application="CarePoint",
            database="ok",
            uptime_seconds=int(time.time() - started),
        ), 200
    except sqlite3.Error:
        APP_STATUS.set(0)
        return jsonify(status="unhealthy", application="CarePoint"), 503


@app.route("/metrics")
def metrics():
    return Response(generate_latest(), mimetype="text/plain; version=0.0.4; charset=utf-8")


if __name__ == "__main__":
    app.run(host="0.0.0.0", port=int(os.environ.get("PORT", 8000)))
