Добре, робимо мінімальний, але правильний сервер на Go: REST, модульна структура, SQL + JSON для гнучких даних. Без зайвої магії, зате з твердим фундаментом 🧱

Я покажу структуру проєкту + код файлів. Це реально можна прямо зараз клонувати в Termux і запустити.


---

1️⃣ Структура проєкту

infoserver/
├── go.mod
├── cmd/
│   └── server/
│       └── main.go
├── internal/
│   ├── config/
│   │   └── config.go
│   ├── db/
│   │   └── db.go
│   ├── model/
│   │   └── record.go
│   ├── repository/
│   │   └── record_repo.go
│   ├── handler/
│   │   └── record_handler.go
│   └── router/
│       └── router.go

Це класична Go-архітектура:

cmd – точка входу

internal – бізнес-логіка

розділення HTTP / DB / модель



---

2️⃣ go.mod

module infoserver

go 1.22

require (
    github.com/go-chi/chi/v5 v5.0.10
    github.com/jackc/pgx/v5 v5.5.4
)


---

3️⃣ Конфіг

internal/config/config.go

package config

import "os"

type Config struct {
    DBUrl string
    Addr  string
}

func Load() Config {
    return Config{
        DBUrl: getEnv("DB_URL", "postgres://user:pass@localhost:5432/info?sslmode=disable"),
        Addr:  getEnv("ADDR", ":8080"),
    }
}

func getEnv(key, def string) string {
    if v := os.Getenv(key); v != "" {
        return v
    }
    return def
}


---

4️⃣ База даних

internal/db/db.go

package db

import (
    "context"
    "github.com/jackc/pgx/v5/pgxpool"
)

func Connect(ctx context.Context, url string) (*pgxpool.Pool, error) {
    return pgxpool.New(ctx, url)
}


---

5️⃣ Модель (SQL + JSON)

internal/model/record.go

package model

import "time"

type Record struct {
    ID        int64                  `json:"id"`
    Date      time.Time              `json:"date"`
    Data      map[string]interface{} `json:"data"`
    CreatedAt time.Time              `json:"created_at"`
}


---

6️⃣ Репозиторій (чистий SQL)

internal/repository/record_repo.go

package repository

import (
    "context"
    "encoding/json"
    "infoserver/internal/model"

    "github.com/jackc/pgx/v5/pgxpool"
)

type RecordRepo struct {
    db *pgxpool.Pool
}

func NewRecordRepo(db *pgxpool.Pool) *RecordRepo {
    return &RecordRepo{db: db}
}

func (r *RecordRepo) Create(ctx context.Context, rec model.Record) error {
    data, _ := json.Marshal(rec.Data)

    _, err := r.db.Exec(ctx,
        `INSERT INTO records (date, data) VALUES ($1, $2)`,
        rec.Date, data,
    )
    return err
}

func (r *RecordRepo) GetByDate(ctx context.Context, date string) ([]model.Record, error) {
    rows, err := r.db.Query(ctx,
        `SELECT id, date, data, created_at FROM records WHERE date = $1`,
        date,
    )
    if err != nil {
        return nil, err
    }
    defer rows.Close()

    var result []model.Record

    for rows.Next() {
        var rec model.Record
        var raw []byte

        if err := rows.Scan(&rec.ID, &rec.Date, &raw, &rec.CreatedAt); err != nil {
            return nil, err
        }

        json.Unmarshal(raw, &rec.Data)
        result = append(result, rec)
    }

    return result, nil
}


---

7️⃣ HTTP handler

internal/handler/record_handler.go

package handler

import (
    "encoding/json"
    "infoserver/internal/model"
    "infoserver/internal/repository"
    "net/http"
    "time"
)

type RecordHandler struct {
    repo *repository.RecordRepo
}

func NewRecordHandler(r *repository.RecordRepo) *RecordHandler {
    return &RecordHandler{repo: r}
}

func (h *RecordHandler) Create(w http.ResponseWriter, r *http.Request) {
    var payload struct {
        Date string                 `json:"date"`
        Data map[string]interface{} `json:"data"`
    }

    json.NewDecoder(r.Body).Decode(&payload)

    date, _ := time.Parse("2006-01-02", payload.Date)

    rec := model.Record{
        Date: date,
        Data: payload.Data,
    }

    if err := h.repo.Create(r.Context(), rec); err != nil {
        http.Error(w, err.Error(), 500)
        return
    }

    w.WriteHeader(http.StatusCreated)
}

func (h *RecordHandler) GetByDate(w http.ResponseWriter, r *http.Request) {
    date := r.URL.Query().Get("date")

    records, err := h.repo.GetByDate(r.Context(), date)
    if err != nil {
        http.Error(w, err.Error(), 500)
        return
    }

    json.NewEncoder(w).Encode(records)
}


---

8️⃣ Роутер

internal/router/router.go

package router

import (
    "infoserver/internal/handler"

    "github.com/go-chi/chi/v5"
)

func New(h *handler.RecordHandler) *chi.Mux {
    r := chi.NewRouter()

    r.Post("/records", h.Create)
    r.Get("/records", h.GetByDate)

    return r
}


---

9️⃣ main.go

cmd/server/main.go

package main

import (
    "context"
    "infoserver/internal/config"
    "infoserver/internal/db"
    "infoserver/internal/handler"
    "infoserver/internal/repository"
    "infoserver/internal/router"
    "log"
    "net/http"
)

func main() {
    cfg := config.Load()
    ctx := context.Background()

    pool, err := db.Connect(ctx, cfg.DBUrl)
    if err != nil {
        log.Fatal(err)
    }
    defer pool.Close()

    repo := repository.NewRecordRepo(pool)
    h := handler.NewRecordHandler(repo)
    r := router.New(h)

    log.Println("Server started on", cfg.Addr)
    log.Fatal(http.ListenAndServe(cfg.Addr, r))
}


---

🔟 SQL схема

CREATE TABLE records (
    id SERIAL PRIMARY KEY,
    date DATE NOT NULL,
    data JSONB NOT NULL,
    created_at TIMESTAMP DEFAULT now()
);

CREATE INDEX idx_records_date ON records(date);
CREATE INDEX idx_records_data ON records USING GIN (data);


---

🔥 Що ти отримав

REST-сервер

Go, строгі типи

SQL + JSONB

легко розширювати

готово для Termux

готово для майбутньої оптимізації



---

Наступні логічні кроки

1. 🔢 calculated fields (на льоту або materialized)


2. 🔍 повнотекстовий пошук


3. 🧠 кешування


4. 🐍 простий Python-клієнт


5. 🧪 міграція JSON → SQL полів



Скажи, що робимо далі.
Я готовий 🔧
