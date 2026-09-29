# trabalo-de-desenvolvimento-de-ssistemas-aplicado-a-dados

!pip install -q fastapi "uvicorn[standard]" sqlalchemy pydantic PyJWT python-multipart requests

# ===========================================================================
# API REST - Controle de Tarefas e Hábitos (FastAPI + SQLAlchemy + JWT)
# Versão de célula única para Google Colab
# ===========================================================================
import hashlib
import hmac
import json
import os
import secrets
import threading
import time
from contextlib import asynccontextmanager
from datetime import date, datetime, timedelta, timezone
from typing import Literal, Optional

import jwt
import requests
import uvicorn
from fastapi import Depends, FastAPI, HTTPException, Query, Response, status
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm
from pydantic import BaseModel, ConfigDict, Field
from sqlalchemy import (
    Boolean, Date, DateTime, ForeignKey, Integer, String, Text,
    UniqueConstraint, create_engine, func, select,
)
from sqlalchemy.orm import (
    DeclarativeBase, Mapped, Session, mapped_column, relationship, sessionmaker,
)

# ---------------------------------------------------------------------------
# Configuração
# ---------------------------------------------------------------------------
DATABASE_URL = os.getenv("DATABASE_URL", "sqlite:///./tracker.db")
SECRET_KEY = os.getenv("SECRET_KEY") or secrets.token_hex(32)
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 60

engine = create_engine(
    DATABASE_URL,
    connect_args={"check_same_thread": False} if DATABASE_URL.startswith("sqlite") else {},
)
SessionLocal = sessionmaker(bind=engine, autoflush=False, expire_on_commit=False)


class Base(DeclarativeBase):
    pass


def utcnow() -> datetime:
    return datetime.now(timezone.utc)


# ---------------------------------------------------------------------------
# Modelos (banco de dados)
# ---------------------------------------------------------------------------
class User(Base):
    __tablename__ = "users"
    id: Mapped[int] = mapped_column(primary_key=True)
    username: Mapped[str] = mapped_column(String(50), unique=True, index=True)
    password_hash: Mapped[str] = mapped_column(String(255))
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=utcnow)
    tasks: Mapped[list["Task"]] = relationship(back_populates="owner", cascade="all, delete-orphan")
    habits: Mapped[list["Habit"]] = relationship(back_populates="owner", cascade="all, delete-orphan")


class Task(Base):
    __tablename__ = "tasks"
    id: Mapped[int] = mapped_column(primary_key=True)
    user_id: Mapped[int] = mapped_column(ForeignKey("users.id"), index=True)
    title: Mapped[str] = mapped_column(String(200))
    description: Mapped[Optional[str]] = mapped_column(Text, nullable=True)
    priority: Mapped[str] = mapped_column(String(10), default="medium")
    status: Mapped[str] = mapped_column(String(15), default="pending")
    due_date: Mapped[Optional[date]] = mapped_column(Date, nullable=True)
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=utcnow)
    completed_at: Mapped[Optional[datetime]] = mapped_column(DateTime(timezone=True), nullable=True)
    owner: Mapped["User"] = relationship(back_populates="tasks")


class Habit(Base):
    __tablename__ = "habits"
    id: Mapped[int] = mapped_column(primary_key=True)
    user_id: Mapped[int] = mapped_column(ForeignKey("users.id"), index=True)
    name: Mapped[str] = mapped_column(String(120))
    description: Mapped[Optional[str]] = mapped_column(Text, nullable=True)
    target_per_week: Mapped[int] = mapped_column(Integer, default=7)
    active: Mapped[bool] = mapped_column(Boolean, default=True)
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), default=utcnow)
    owner: Mapped["User"] = relationship(back_populates="habits")
    logs: Mapped[list["HabitLog"]] = relationship(back_populates="habit", cascade="all, delete-orphan")


class HabitLog(Base):
    __tablename__ = "habit_logs"
    __table_args__ = (UniqueConstraint("habit_id", "log_date", name="uq_habit_day"),)
    id: Mapped[int] = mapped_column(primary_key=True)
    habit_id: Mapped[int] = mapped_column(ForeignKey("habits.id"), index=True)
    log_date: Mapped[date] = mapped_column(Date)
    note: Mapped[Optional[str]] = mapped_column(String(300), nullable=True)
    habit: Mapped["Habit"] = relationship(back_populates="logs")


# ---------------------------------------------------------------------------
# Schemas (validação de entrada/saída)
# ---------------------------------------------------------------------------
Priority = Literal["low", "medium", "high"]
TaskStatus = Literal["pending", "in_progress", "done"]


class UserCreate(BaseModel):
    username: str = Field(min_length=3, max_length=50, pattern=r"^[A-Za-z0-9_.-]+$")
    password: str = Field(min_length=6, max_length=128)


class UserOut(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    id: int
    username: str
    created_at: datetime


class Token(BaseModel):
    access_token: str
    token_type: str = "bearer"


class TaskCreate(BaseModel):
    title: str = Field(min_length=1, max_length=200)
    description: Optional[str] = None
    priority: Priority = "medium"
    status: TaskStatus = "pending"
    due_date: Optional[date] = None


class TaskUpdate(BaseModel):
    title: Optional[str] = Field(default=None, min_length=1, max_length=200)
    description: Optional[str] = None
    priority: Optional[Priority] = None
    status: Optional[TaskStatus] = None
    due_date: Optional[date] = None


class TaskOut(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    id: int
    title: str
    description: Optional[str]
    priority: str
    status: str
    due_date: Optional[date]
    created_at: datetime
    completed_at: Optional[datetime]


class HabitCreate(BaseModel):
    name: str = Field(min_length=1, max_length=120)
    description: Optional[str] = None
    target_per_week: int = Field(default=7, ge=1, le=7)


class HabitUpdate(BaseModel):
    name: Optional[str] = Field(default=None, min_length=1, max_length=120)
    description: Optional[str] = None
    target_per_week: Optional[int] = Field(default=None, ge=1, le=7)
    active: Optional[bool] = None


class HabitOut(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    id: int
    name: str
    description: Optional[str]
    target_per_week: int
    active: bool
    created_at: datetime


class CheckinCreate(BaseModel):
    log_date: Optional[date] = None
    note: Optional[str] = Field(default=None, max_length=300)


class HabitLogOut(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    id: int
    habit_id: int
    log_date: date
    note: Optional[str]


class HabitStats(BaseModel):
    habit_id: int
    total_checkins: int
    current_streak: int
    best_streak: int
    checkins_last_30_days: int
    completion_rate_30_days: float
    done_today: bool


# ---------------------------------------------------------------------------
# Segurança
# ---------------------------------------------------------------------------
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="auth/login")


def hash_password(password: str) -> str:
    salt = secrets.token_bytes(16)
    digest = hashlib.pbkdf2_hmac("sha256", password.encode(), salt, 200_000)
    return f"{salt.hex()}${digest.hex()}"


def verify_password(password: str, stored: str) -> bool:
    try:
        salt_hex, digest_hex = stored.split("$")
        digest = hashlib.pbkdf2_hmac("sha256", password.encode(), bytes.fromhex(salt_hex), 200_000)
        return hmac.compare_digest(digest.hex(), digest_hex)
    except ValueError:
        return False


def create_access_token(user_id: int) -> str:
    payload = {"sub": str(user_id), "exp": utcnow() + timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)}
    return jwt.encode(payload, SECRET_KEY, algorithm=ALGORITHM)


def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()


def get_current_user(token: str = Depends(oauth2_scheme), db: Session = Depends(get_db)) -> User:
    credentials_error = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Credenciais inválidas ou token expirado",
        headers={"WWW-Authenticate": "Bearer"},
    )
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        user_id = int(payload["sub"])
    except (jwt.PyJWTError, KeyError, ValueError):
        raise credentials_error
    user = db.get(User, user_id)
    if not user:
        raise credentials_error
    return user


# ---------------------------------------------------------------------------
# Funções auxiliares
# ---------------------------------------------------------------------------
def get_owned_task(db: Session, task_id: int, user: User) -> Task:
    task = db.get(Task, task_id)
    if not task or task.user_id != user.id:
        raise HTTPException(status_code=404, detail="Tarefa não encontrada")
    return task


def get_owned_habit(db: Session, habit_id: int, user: User) -> Habit:
    habit = db.get(Habit, habit_id)
    if not habit or habit.user_id != user.id:
        raise HTTPException(status_code=404, detail="Hábito não encontrado")
    return habit


def calc_streaks(days: set[date]) -> tuple[int, int]:
    """Retorna (sequência atual, melhor sequência)."""
    if not days:
        return 0, 0
    ordered = sorted(days)
    best = run = 1
    for prev, cur in zip(ordered, ordered[1:]):
        run = run + 1 if (cur - prev).days == 1 else 1
        best = max(best, run)
    today = date.today()
    cursor = today if today in days else today - timedelta(days=1)
    current = 0
    while cursor in days:
        current += 1
        cursor -= timedelta(days=1)
    return current, best


# ---------------------------------------------------------------------------
# Aplicação
# ---------------------------------------------------------------------------
@asynccontextmanager
async def lifespan(_: FastAPI):
    Base.metadata.create_all(bind=engine)
    yield


app = FastAPI(
    title="API de Tarefas e Hábitos",
    description="Controle de tarefas (to-do) e acompanhamento de hábitos com streaks.",
    version="1.0.0",
    lifespan=lifespan,
)


@app.get("/", tags=["Sistema"])
def root():
    return {"status": "ok", "docs": "/docs"}


# ------------------------------- Autenticação ------------------------------
@app.post("/auth/register", response_model=UserOut, status_code=201, tags=["Autenticação"])
def register(data: UserCreate, db: Session = Depends(get_db)):
    if db.scalar(select(User).where(User.username == data.username)):
        raise HTTPException(status_code=409, detail="Nome de usuário já em uso")
    user = User(username=data.username, password_hash=hash_password(data.password))
    db.add(user)
    db.commit()
    return user


@app.post("/auth/login", response_model=Token, tags=["Autenticação"])
def login(form: OAuth2PasswordRequestForm = Depends(), db: Session = Depends(get_db)):
    user = db.scalar(select(User).where(User.username == form.username))
    if not user or not verify_password(form.password, user.password_hash):
        raise HTTPException(status_code=401, detail="Usuário ou senha incorretos")
    return Token(access_token=create_access_token(user.id))


@app.get("/auth/me", response_model=UserOut, tags=["Autenticação"])
def me(user: User = Depends(get_current_user)):
    return user


# --------------------------------- Tarefas ---------------------------------
@app.post("/tasks", response_model=TaskOut, status_code=201, tags=["Tarefas"])
def create_task(data: TaskCreate, db: Session = Depends(get_db), user: User = Depends(get_current_user)):
    task = Task(**data.model_dump(), user_id=user.id)
    if task.status == "done":
        task.completed_at = utcnow()
    db.add(task)
    db.commit()
    return task


@app.get("/tasks", response_model=list[TaskOut], tags=["Tarefas"])
def list_tasks(
    status_: Optional[TaskStatus] = Query(default=None, alias="status"),
    priority: Optional[Priority] = None,
    due_before: Optional[date] = None,
    overdue: bool = False,
    search: Optional[str] = Query(default=None, min_length=1),
    skip: int = Query(default=0, ge=0),
    limit: int = Query(default=50, ge=1, le=200),
    db: Session = Depends(get_db),
    user: User = Depends(get_current_user),
):
    query = select(Task).where(Task.user_id == user.id)
    if status_:
        query = query.where(Task.status == status_)
    if priority:
        query = query.where(Task.priority == priority)
    if due_before:
        query = query.where(Task.due_date <= due_before)
    if overdue:
        query = query.where(Task.due_date < date.today(), Task.status != "done")
    if search:
        like = f"%{search}%"
        query = query.where(Task.title.ilike(like) | Task.description.ilike(like))
    query = query.order_by(Task.due_date.is_(None), Task.due_date, Task.id).offset(skip).limit(limit)
    return db.scalars(query).all()


@app.get("/tasks/{task_id}", response_model=TaskOut, tags=["Tarefas"])
def get_task(task_id: int, db: Session = Depends(get_db), user: User = Depends(get_current_user)):
    return get_owned_task(db, task_id, user)


@app.patch("/tasks/{task_id}", response_model=TaskOut, tags=["Tarefas"])
def update_task(task_id: int, data: TaskUpdate, db: Session = Depends(get_db),
                user: User = Depends(get_current_user)):
    task = get_owned_task(db, task_id, user)
    changes = data.model_dump(exclude_unset=True)
    for field, value in changes.items():
        setattr(task, field, value)
    if "status" in changes:
        task.completed_at = utcnow() if task.status == "done" else None
    db.commit()
    return task


@app.post("/tasks/{task_id}/complete", response_model=TaskOut, tags=["Tarefas"])
def complete_task(task_id: int, db: Session = Depends(get_db), user: User = Depends(get_current_user)):
    task = get_owned_task(db, task_id, user)
    task.status = "done"
    task.completed_at = utcnow()
    db.commit()
    return task


@app.delete("/tasks/{task_id}", status_code=204, tags=["Tarefas"])
def delete_task(task_id: int, db: Session = Depends(get_db), user: User = Depends(get_current_user)):
    task = get_owned_task(db, task_id, user)
    db.delete(task)
    db.commit()
    return Response(status_code=204)


# --------------------------------- Hábitos ---------------------------------
@app.post("/habits", response_model=HabitOut, status_code=201, tags=["Hábitos"])
def create_habit(data: HabitCreate, db: Session = Depends(get_db), user: User = Depends(get_current_user)):
    habit = Habit(**data.model_dump(), user_id=user.id)
    db.add(habit)
    db.commit()
    return habit


@app.get("/habits", response_model=list[HabitOut], tags=["Hábitos"])
def list_habits(active: Optional[bool] = None, db: Session = Depends(get_db),
                user: User = Depends(get_current_user)):
    query = select(Habit).where(Habit.user_id == user.id)
    if active is not None:
        query = query.where(Habit.active == active)
    return db.scalars(query.order_by(Habit.id)).all()


@app.get("/habits/{habit_id}", response_model=HabitOut, tags=["Hábitos"])
def get_habit(habit_id: int, db: Session = Depends(get_db), user: User = Depends(get_current_user)):
    return get_owned_habit(db, habit_id, user)


@app.patch("/habits/{habit_id}", response_model=HabitOut, tags=["Hábitos"])
def update_habit(habit_id: int, data: HabitUpdate, db: Session = Depends(get_db),
                 user: User = Depends(get_current_user)):
    habit = get_owned_habit(db, habit_id, user)
    for field, value in data.model_dump(exclude_unset=True).items():
        setattr(habit, field, value)
    db.commit()
    return habit


@app.delete("/habits/{habit_id}", status_code=204, tags=["Hábitos"])
def delete_habit(habit_id: int, db: Session = Depends(get_db), user: User = Depends(get_current_user)):
    habit = get_owned_habit(db, habit_id, user)
    db.delete(habit)
    db.commit()
    return Response(status_code=204)


@app.post("/habits/{habit_id}/checkin", response_model=HabitLogOut, status_code=201, tags=["Hábitos"])
def checkin(habit_id: int, data: Optional[CheckinCreate] = None, db: Session = Depends(get_db),
            user: User = Depends(get_current_user)):
    habit = get_owned_habit(db, habit_id, user)
    data = data or CheckinCreate()
    day = data.log_date or date.today()
    if day > date.today():
        raise HTTPException(status_code=422, detail="Não é possível registrar check-in no futuro")
    if db.scalar(select(HabitLog).where(HabitLog.habit_id == habit.id, HabitLog.log_date == day)):
        raise HTTPException(status_code=409, detail="Check-in já registrado para essa data")
    log = HabitLog(habit_id=habit.id, log_date=day, note=data.note)
    db.add(log)
    db.commit()
    return log


@app.get("/habits/{habit_id}/logs", response_model=list[HabitLogOut], tags=["Hábitos"])
def list_logs(habit_id: int, start: Optional[date] = None, end: Optional[date] = None,
              db: Session = Depends(get_db), user: User = Depends(get_current_user)):
    habit = get_owned_habit(db, habit_id, user)
    query = select(HabitLog).where(HabitLog.habit_id == habit.id)
    if start:
        query = query.where(HabitLog.log_date >= start)
    if end:
        query = query.where(HabitLog.log_date <= end)
    return db.scalars(query.order_by(HabitLog.log_date.desc())).all()


@app.delete("/habits/{habit_id}/checkin/{log_date}", status_code=204, tags=["Hábitos"])
def remove_checkin(habit_id: int, log_date: date, db: Session = Depends(get_db),
                   user: User = Depends(get_current_user)):
    habit = get_owned_habit(db, habit_id, user)
    log = db.scalar(select(HabitLog).where(HabitLog.habit_id == habit.id, HabitLog.log_date == log_date))
    if not log:
        raise HTTPException(status_code=404, detail="Check-in não encontrado")
    db.delete(log)
    db.commit()
    return Response(status_code=204)


@app.get("/habits/{habit_id}/stats", response_model=HabitStats, tags=["Hábitos"])
def habit_stats(habit_id: int, db: Session = Depends(get_db), user: User = Depends(get_current_user)):
    habit = get_owned_habit(db, habit_id, user)
    days = set(db.scalars(select(HabitLog.log_date).where(HabitLog.habit_id == habit.id)).all())
    current, best = calc_streaks(days)
    today = date.today()
    since = today - timedelta(days=29)
    last_30 = sum(1 for d in days if since <= d <= today)
    expected = habit.target_per_week * 30 / 7
    rate = min(1.0, last_30 / expected) if expected else 0.0
    return HabitStats(
        habit_id=habit.id, total_checkins=len(days), current_streak=current, best_streak=best,
        checkins_last_30_days=last_30, completion_rate_30_days=round(rate, 3),
        done_today=today in days,
    )


# -------------------------------- Dashboard --------------------------------
@app.get("/dashboard", tags=["Resumo"])
def dashboard(db: Session = Depends(get_db), user: User = Depends(get_current_user)):
    rows = db.execute(
        select(Task.status, func.count()).where(Task.user_id == user.id).group_by(Task.status)
    ).all()
    tasks_by_status = {"pending": 0, "in_progress": 0, "done": 0}
    tasks_by_status.update({s: c for s, c in rows})
    overdue = db.scalar(
        select(func.count()).where(
            Task.user_id == user.id, Task.due_date < date.today(), Task.status != "done"
        )
    )
    active_habits = db.scalars(
        select(Habit).where(Habit.user_id == user.id, Habit.active.is_(True))
    ).all()
    done_today_ids = set(
        db.scalars(
            select(HabitLog.habit_id).where(
                HabitLog.habit_id.in_([h.id for h in active_habits] or [0]),
                HabitLog.log_date == date.today(),
            )
        ).all()
    )
    return {
        "tasks": {**tasks_by_status, "overdue": overdue},
        "habits": {
            "active": len(active_habits),
            "done_today": len(done_today_ids),
            "pending_today": [
                {"id": h.id, "name": h.name} for h in active_habits if h.id not in done_today_ids
            ],
        },
    }


# ===========================================================================
# SOBE O SERVIDOR EM SEGUNDO PLANO (dentro do próprio Colab)
# ===========================================================================
BASE = "http://127.0.0.1:8000"

try:  # se a célula for executada de novo, derruba o servidor anterior
    server.should_exit = True
    time.sleep(2)
except NameError:
    pass

server = uvicorn.Server(uvicorn.Config(app, host="127.0.0.1", port=8000, log_level="warning"))
threading.Thread(target=server.run, daemon=True).start()
while not server.started:
    time.sleep(0.1)
print("✅ API rodando em", BASE)


# ===========================================================================
# DEMONSTRAÇÃO: chamando a API como um cliente faria
# ===========================================================================
def show(titulo, r):
    print(f"\n=== {titulo} [HTTP {r.status_code}] ===")
    if r.content:
        print(json.dumps(r.json(), indent=2, ensure_ascii=False))


usuario = "aluno_" + secrets.token_hex(3)
senha = "senha123"

show("Cadastro", requests.post(f"{BASE}/auth/register", json={"username": usuario, "password": senha}))

r = requests.post(f"{BASE}/auth/login", data={"username": usuario, "password": senha})
token = r.json()["access_token"]
H = {"Authorization": f"Bearer {token}"}
print("\n=== Login OK, token JWT obtido ===")

# --- Tarefas ---
amanha = (date.today() + timedelta(days=1)).isoformat()
r = requests.post(f"{BASE}/tasks", headers=H, json={
    "title": "Entregar trabalho da faculdade", "priority": "high", "due_date": amanha})
show("Criar tarefa", r)
tarefa_id = r.json()["id"]

requests.post(f"{BASE}/tasks", headers=H, json={"title": "Estudar Python", "priority": "medium"})
show("Listar tarefas", requests.get(f"{BASE}/tasks", headers=H))
show("Concluir tarefa", requests.post(f"{BASE}/tasks/{tarefa_id}/complete", headers=H))

# --- Hábitos ---
r = requests.post(f"{BASE}/habits", headers=H, json={
    "name": "Beber 2L de água", "target_per_week": 7})
show("Criar hábito", r)
habito_id = r.json()["id"]

ontem = (date.today() - timedelta(days=1)).isoformat()
anteontem = (date.today() - timedelta(days=2)).isoformat()
requests.post(f"{BASE}/habits/{habito_id}/checkin", headers=H, json={"log_date": anteontem})
requests.post(f"{BASE}/habits/{habito_id}/checkin", headers=H, json={"log_date": ontem})
show("Check-in de hoje", requests.post(
    f"{BASE}/habits/{habito_id}/checkin", headers=H, json={"note": "Fiz hoje!"}))

show("Estatísticas do hábito", requests.get(f"{BASE}/habits/{habito_id}/stats", headers=H))
show("Dashboard", requests.get(f"{BASE}/dashboard", headers=H))
