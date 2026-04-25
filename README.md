# 📚 Biblioteca De los Ecos — Backend Setup Guide

## Arquitectura

```
Frontend (React)
    │
    │  signInWithEmailAndPassword()
    ▼
Firebase Auth ──── ID Token ────►  FastAPI Backend
                                        │
                                        │  firebase-admin verify_id_token()
                                        │  Role check via Firestore
                                        ▼
                                   Firestore DB
                                   Firebase Storage
```

## Estructura de archivos

```
biblioteca/
├── backend/
│   ├── main.py                  # FastAPI app, CORS, routers
│   ├── config.py                # Firebase Admin SDK init
│   ├── models.py                # Pydantic models
│   ├── middleware/
│   │   └── auth.py              # Token verification, role guards
│   └── routers/
│       ├── auth.py              # /auth/register, /auth/me, /auth/logout
│       ├── books.py             # /books/ CRUD + portadas
│       ├── loans.py             # /loans/ ciclo completo
│       ├── community.py         # /community/posts + comentarios + likes
│       └── admin.py             # /admin/stats, /admin/users, /admin/purchases
├── firebase/
│   ├── firestore.rules          # Seguridad Firestore (deploy con Firebase CLI)
│   ├── storage.rules            # Seguridad Storage
│   └── firestore.indexes.json   # Índices compuestos para queries
├── frontend/
│   └── services/
│       └── api.js               # Capa de servicio: Firebase Auth + llamadas FastAPI
├── requirements.txt
└── .env.example                 # Plantilla de variables de entorno
```

---

## 1. Configurar Firebase

### 1.1 Crear proyecto

1. Ve a [console.firebase.google.com](https://console.firebase.google.com)
2. Crea un nuevo proyecto (ej. `bibliotheca-arcana`)
3. Activa **Firestore Database** en modo producción
4. Activa **Authentication** → Email/Password
5. Activa **Storage**

### 1.2 Service Account (backend)

1. Firebase Console → Configuración del proyecto → **Cuentas de servicio**
2. Haz clic en **Generar nueva clave privada**
3. Descarga el JSON y guárdalo como `firebase/serviceAccountKey.json`
4. **NUNCA subas este archivo a git** (ya está en `.gitignore`)

### 1.3 Configuración del cliente (frontend)

1. Firebase Console → Configuración del proyecto → **Tus aplicaciones** → Añadir app web
2. Copia los valores al `.env` del frontend:

```env
VITE_FIREBASE_API_KEY=AIzaSy...
VITE_FIREBASE_AUTH_DOMAIN=biblioteca-xxxxx.firebaseapp.com
VITE_FIREBASE_PROJECT_ID=biblioteca-xxxxx
VITE_FIREBASE_STORAGE_BUCKET=biblioteca-xxxxx.appspot.com
VITE_FIREBASE_MESSAGING_SENDER_ID=123456789
VITE_FIREBASE_APP_ID=1:123456789:web:abcdef
VITE_API_URL=http://localhost:8000/api/v1
```

---

## 2. Configurar el backend

### 2.1 Entorno virtual e instalación

```bash
python -m venv .venv
#  source .venv/bin/activate     # Linux/Mac
.venv\Scripts\activate           # Windows

pip install -r requirements.txt
```

### 2.2 Variables de entorno

```bash
cp .env.example .env
# Edita .env con tus valores reales
```

### 2.3 Ejecutar en desarrollo

```bash
uvicorn backend.main:app --reload --host 0.0.0.0 --port 8000
```

Visita **http://localhost:8000/docs** para la documentación interactiva (Swagger UI).

---

## 3. Desplegar reglas de Firestore

```bash
# Instalar Firebase CLI si no lo tienes
npm install -g firebase-tools

firebase login
firebase use bibliotheca-arcana-xxxxx

# Desplegar reglas e índices
firebase deploy --only firestore:rules,firestore:indexes
firebase deploy --only storage
```

---

## 4. Colecciones de Firestore

| Colección           | Documentos          | Notas                                    |
|---------------------|---------------------|------------------------------------------|
| `users`             | `{uid}`             | UID de Firebase Auth como ID de documento |
| `books`             | `{uuid}`            | Creados por el backend                   |
| `loans`             | `{uuid}`            | Transacciones atómicas al aprobar        |
| `community_posts`   | `{uuid}`            | Subcolecciones: `likes`, `comments`      |
| `purchases`         | `{uuid}`            | Compras de ebooks                        |

---

## 5. Flujo de autenticación paso a paso

```
1. Usuario escribe email + contraseña en el frontend

2. Frontend llama a Firebase Auth SDK:
   signInWithEmailAndPassword(auth, email, password)
   → Firebase devuelve un ID Token (JWT, válido 1 hora)

3. Frontend llama a /api/v1/auth/me con:
   Authorization: Bearer <ID_Token>

4. FastAPI recibe el token y llama a:
   firebase_admin.auth.verify_id_token(token)
   → Si es válido, devuelve el decoded token con el UID

5. FastAPI carga el documento users/{uid} de Firestore
   → Obtiene el rol y otros datos del perfil

6. El endpoint devuelve el perfil del usuario

7. Para cada request posterior, el frontend repite el paso 3
   (Firebase SDK renueva el token automáticamente antes de que expire)
```

---

## 6. Sistema de roles

| Rol         | Nombre en UI            | Permisos                                      |
|-------------|-------------------------|-----------------------------------------------|
| `student`   | Aprendiz de Letras      | Leer catálogo, solicitar préstamos, comunidad |
| `librarian` | Custodio de Volúmenes   | + CRUD libros, gestionar préstamos, moderar   |
| `admin`     | Archimago del Sistema   | + Estadísticas, gestión de usuarios, config   |

Los roles se asignan en Firestore y se sincronizan como **Custom Claims** en Firebase Auth.
Para cambiar el rol de un usuario, un admin usa `PATCH /admin/users/{uid}`.

---

## 7. Producción

### Opciones de despliegue para FastAPI

| Plataforma         | Comando / Notas                                            |
|--------------------|------------------------------------------------------------|
| **Cloud Run**      | `gcloud run deploy` — escala a cero automáticamente        |
| **Railway**        | `railway up` — fácil y barato para proyectos escolares     |
| **Render**         | `gunicorn backend.main:app -k uvicorn.workers.UvicornWorker` |
| **VPS (Ubuntu)**   | `systemd` + `nginx` como proxy inverso                    |

### Check de vencimientos automático

Configura un **Cloud Scheduler** que llame a `POST /api/v1/loans/check-overdue` 
con un token de admin cada noche a las 00:00:

```
Cron: 0 0 * * *
URL: https://tu-api.com/api/v1/loans/check-overdue
Auth: Bearer <admin_token>
```

---

## 8. Endpoints resumen

```
POST   /api/v1/auth/register
GET    /api/v1/auth/me
PATCH  /api/v1/auth/me
POST   /api/v1/auth/logout

GET    /api/v1/books/                     ?search &category &available &book_type
POST   /api/v1/books/                     [librarian]
GET    /api/v1/books/{id}
PATCH  /api/v1/books/{id}                 [librarian]
DELETE /api/v1/books/{id}                 [librarian]
POST   /api/v1/books/{id}/cover           [librarian] multipart
GET    /api/v1/books/meta/categories

GET    /api/v1/loans/                     ?status
POST   /api/v1/loans/
PATCH  /api/v1/loans/{id}                 [librarian] approve/deny
POST   /api/v1/loans/{id}/return          [librarian]
POST   /api/v1/loans/check-overdue        [admin]

GET    /api/v1/community/posts            ?book_tag
POST   /api/v1/community/posts
PATCH  /api/v1/community/posts/{id}
DELETE /api/v1/community/posts/{id}
POST   /api/v1/community/posts/{id}/like
GET    /api/v1/community/posts/{id}/comments
POST   /api/v1/community/posts/{id}/comments
DELETE /api/v1/community/posts/{id}/comments/{cid}

GET    /api/v1/admin/stats                [admin]
GET    /api/v1/admin/users                [admin]
PATCH  /api/v1/admin/users/{uid}          [admin]
DELETE /api/v1/admin/users/{uid}          [admin]
POST   /api/v1/admin/purchases
GET    /api/v1/admin/purchases/my

GET    /health
```
