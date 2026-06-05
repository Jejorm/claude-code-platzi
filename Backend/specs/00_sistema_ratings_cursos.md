# Análisis de Impacto — Sistema de Ratings de Cursos (1–5 estrellas)

## Contexto

PlatziFlix necesita una funcionalidad de calificación de cursos: un curso puede
recibir un rating de 1 a 5 estrellas.

Este es un **proyecto de estudio sin usuarios reales**, por lo que el objetivo es
ejercitar el slice vertical completo en todas las capas siguiendo las convenciones
existentes — no endurecer el sistema contra abuso.

### Decisión arquitectónica (elegida): Opción A — tabla `ratings` anónima

Una nueva tabla `ratings` (sin `user_id`) almacena una fila por cada calificación
enviada. El promedio y el conteo del curso se calculan con `AVG`/`COUNT`. Esto
replica el patrón de entidades existente (`Course`/`Teacher`/`Lesson`) y obliga a
recorrer el camino completo de lectura + escritura en cada cliente.

**Limitación aceptada y conocida:** sin autenticación (gap #1 documentado), nada
impide que la misma persona vote múltiples veces. Es aceptable para un proyecto de
estudio; se registra como limitación y no se construye un sistema de usuarios para
evitarlo.

---

## Impacto por Componente

### 1. Backend (FastAPI) — fuente de verdad, se hace primero

Sigue el patrón de 3 capas existente (Routes → Service → ORM, sin repositorio).

- **Modelo** — nuevo `app/models/rating.py`: `Rating(BaseModel)` con `course_id`
  (`ForeignKey("courses.id")`, indexado) y `score` (`Integer`). Hereda
  `id`/`created_at`/`updated_at`/`deleted_at` de `BaseModel` (`app/models/base.py`).
  Opcionalmente agregar `relationship` hacia `Course`. Registrarlo en
  `app/models/__init__.py` (import + `__all__`) para que Alembic lo detecte.
- **Migración** — ejecutar `make create-migration` (autogenerate). Verificar que el
  archivo generado en `app/alembic/versions/` siga la convención existente:
  `op.create_index(op.f('ix_ratings_...'))`, `deleted_at` nullable, `downgrade` en
  orden inverso.
- **Servicio** — nuevo `app/services/rating_service.py`: `RatingService(db: Session)`
  alineado con `course_service.py`. Los métodos devuelven dicts planos:
  - `create_rating(slug, score)` — resolver curso por slug (filtrando
    `deleted_at.is_(None)`), validar `1 <= score <= 5`, insertar fila, devolver el
    agregado nuevo.
  - `get_course_rating(slug)` — devolver `{ "rating_avg": float, "rating_count": int }`
    usando `func.avg`/`func.count` filtrando por `deleted_at.is_(None)`.
- **Enriquecer detalle de curso** — en
  `app/services/course_service.py::get_course_by_slug`, agregar `rating_avg` y
  `rating_count` al dict devuelto, de modo que el endpoint de detalle ya traiga el
  rating (los clientes lo leen desde un solo lugar).
- **Rutas** — en `app/main.py`, agregar una dependencia `get_rating_service` (mismo
  patrón `Depends(get_db)` que `get_course_service`) y dos handlers siguiendo el
  estilo existente:
  - `POST /courses/{slug}/ratings` — body `{ "score": int }`, 404 si el curso no
    existe, devuelve el agregado nuevo.
  - `GET /courses/{slug}/ratings` — devuelve `{ rating_avg, rating_count }`.
- **Schemas Pydantic** — el proyecto HOY no tiene schemas. Esta funcionalidad
  introduce el primer body de request, así que crear `app/schemas/rating.py` con
  `RatingCreate` (`score: int`, validado 1–5) y `RatingResponse`. Es el lugar
  correcto para imponer el rango 1–5. (Alternativa: validar inline en el servicio
  para mantenerse sin schemas, pero un schema es lo apropiado para un body POST.)
- **Seed** — extender `app/db/seed.py`: agregar algunas filas `Rating` por curso en
  `create_sample_data()`, y `db.query(Rating).delete()` al inicio de
  `clear_all_data()` (antes de los cursos, para respetar el FK).
- **Contrato** — extender `Backend/specs/00_contracts.md` con un bloque de entidad
  `Rating` y los dos bloques de endpoint nuevos, en el estilo JSON minimalista
  existente. Anotar acá la limitación de voto múltiple sin auth.

### 2. Frontend (Next.js 15)

- **Tipo** — agregar `Rating` a `src/types/index.ts`; agregar `rating_avg?: number`
  y `rating_count?: number` a `CourseDetail`.
- **Componente StarRating** — nuevo `src/components/StarRating/` siguiendo la
  convención de 3 partes existente (`.tsx` + `.module.scss` importando
  `../../styles/vars.scss`). Estrella llena `color('primary')` (`#ff2d2d`), vacía
  `color('light-gray')`. Construirlo con soporte para modo read-only (mostrar
  promedio) y modo interactivo (enviar voto).
- **Display** — renderizar las estrellas read-only del promedio dentro de
  `src/components/CourseDetail/CourseDetail.tsx`, en el bloque `.stats` existente,
  junto a duración / cantidad de clases. El dato ya llega vía el fetch enriquecido
  `getCourseData` en `src/app/course/[slug]/page.tsx` — no hace falta un fetch nuevo
  para el display.
- **Submit** — el control interactivo necesita `useState` + handler de click, así
  que debe ser un componente `"use client"` (un `RatingForm`) que haga POST a
  `http://localhost:8000/courses/{slug}/ratings`. Hoy no hay abstracción de cliente
  API — hacer el POST inline como las llamadas `fetch` existentes (aceptable para
  estudio; opcionalmente introducir un `src/lib/api.ts` mínimo si se quiere enseñar
  ese refactor).
- **Test** — agregar `StarRating.test.tsx` siguiendo el patrón de render de
  `Course.test.tsx` (globals de Vitest, `@testing-library/react`).

> Nota: el bug preexistente de campos `name`/`title` y `teacher_id`/`teacher`
> (gap #2) queda **fuera de alcance** de esta funcionalidad — no corregirlo acá
> salvo pedido explícito.

### 3. Mobile — iOS (SwiftUI, CLEAR)

iOS ya tiene detalle de curso, Teacher y Class — el cliente más completo.

- **DTO** — `Data/Entities/CourseDTO.swift`: agregar `rating_avg`/`rating_count`
  (`Double?`/`Int?`) con `CodingKeys`. Agregarlos también a `CourseDetailDTO`.
- **Modelo de dominio** — `Domain/Models/Course.swift`: agregar
  `ratingAvg`/`ratingCount` más una propiedad computada `displayRating`.
- **Mapper** — `Data/Mapper/CourseMapper.swift`: pasar los campos nuevos en ambos
  overloads de `toDomain`.
- **Endpoint** — `Data/Repositories/CourseAPIEndpoints.swift`: agregar
  `case submitRating(slug: String)` → `POST /courses/{slug}/ratings`. Reutilizar el
  overload de body existente `NetworkManager.request(_:body:responseType:)`; definir
  un `RatingRequestDTO: Codable { let score: Int }`.
- **Repositorio** — agregar `submitRating(_:forSlug:)` a `CourseRepositoryProtocol`
  e implementarlo en `RemoteCourseRepository`.
- **Vista** — fila de estrellas con SF Symbols (`star.fill`). El `onTap` de la card
  → `viewModel.selectCourse(_:)` hoy es un stub `// TODO: Navigate`; una pantalla de
  detalle completa es una tarea mayor. Mínimo viable: estrellas read-only en
  `CourseCardView`; el UI de submit pertenece a un futuro `CourseDetailView`.

### 4. Mobile — Android (Compose, CLEAR + MVI)

Android es el menos completo: solo `GET /courses`, sin pantalla de detalle, modelos
más livianos.

- **DTO** — `data/entities/CourseDTO.kt`: agregar
  `@SerializedName("rating_avg") val ratingAvg: Double? = null` y `rating_count`.
- **Modelo de dominio** — `domain/models/Course.kt`: agregar `ratingAvg`/`ratingCount`.
- **Mapper** — `data/.../CourseMapper.kt`: pasarlos en `fromDTO`.
- **Display ahora** — un composable `RatingBar` (`Row` de 5
  `Icon(Icons.Filled.Star)` usando tokens de `MaterialTheme.colorScheme` como
  `CourseCard`) mostrando el promedio en la card de la lista existente. Este es el
  slice de bajo esfuerzo y alto valor para Android.
- **Submit (mayor)** — requiere superficie nueva que hoy no existe: pantalla de
  detalle, `getCourseBySlug` en `ApiService`/`CourseRepository`, una llamada
  `submitRating`, un `CourseDetailUiState`/`UiEvent`, un `CourseDetailViewModel` y un
  factory `AppModule.provideCourseDetailViewModel(slug)`. Tratarlo como slice
  posterior, no parte del primer corte. Agregar `rating` a
  `MockCourseRepository.kt` y un `submitRating` no-op.

---

## Secuencia Recomendada

1. **Backend primero** — es la fuente de verdad del contrato. Nada del lado cliente
   se puede verificar hasta que existan los endpoints.
2. **Frontend** — el round trip completo de lectura + escritura más rápido para
   validar la API de punta a punta.
3. **iOS** — agregar el campo + display read-only (submit cuando exista pantalla de
   detalle).
4. **Android** — agregar el campo + `RatingBar` read-only; el submit es un slice
   aparte condicionado a construir la pantalla de detalle.

## Archivos Críticos

- `Backend/app/models/rating.py` (nuevo), `Backend/app/models/__init__.py`
- `Backend/app/services/rating_service.py` (nuevo), `Backend/app/services/course_service.py`
- `Backend/app/main.py`, `Backend/app/schemas/rating.py` (directorio nuevo)
- `Backend/app/db/seed.py`, `Backend/specs/00_contracts.md`
- `Frontend/src/types/index.ts`, `Frontend/src/components/StarRating/` (nuevo),
  `Frontend/src/components/CourseDetail/CourseDetail.tsx`,
  `Frontend/src/app/course/[slug]/page.tsx`
- iOS: `CourseDTO.swift`, `Course.swift`, `CourseMapper.swift`,
  `CourseAPIEndpoints.swift`, `CourseRepository*.swift`
- Android: `CourseDTO.kt`, `Course.kt`, `CourseMapper.kt`, `ApiService.kt`,
  `CourseRepository.kt`, `RemoteCourseRepository.kt`, `AppModule.kt`

## Verificación

- **Backend**: `make start` + `make migrate` + `make seed-fresh`. Luego
  `curl -X POST localhost:8000/courses/<slug>/ratings -d '{"score":4}'`,
  `curl localhost:8000/courses/<slug>/ratings`, y confirmar que
  `GET /courses/<slug>` ahora devuelve `rating_avg`/`rating_count`. Rechazar scores
  fuera de rango (esperar 422).
- **Frontend**: `yarn dev`, abrir `/course/<slug>`, ver las estrellas del promedio;
  enviar un rating, confirmar que el promedio se actualiza tras el refetch.
  `yarn test` para `StarRating`.
- **iOS**: correr en simulador contra `localhost:8000`; confirmar que las estrellas
  renderizan con el promedio del seed.
- **Android**: emulador contra `10.0.2.2:8000`; confirmar que `RatingBar` muestra el
  promedio en la lista. (El camino de submit se verifica recién cuando exista la
  pantalla de detalle.)
