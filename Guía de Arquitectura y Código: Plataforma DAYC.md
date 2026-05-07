# Guía de Arquitectura y Código: Plataforma DAYC-2

Bienvenido a la guía técnica de la **Plataforma DAYC-2**, un sistema integral para la evaluación del desarrollo infantil temprano. Esta guía está diseñada para que puedas comprender a la perfección cómo está estructurado el código, qué tecnologías se utilizan, y cómo se comunican todas las partes del sistema. Si en el futuro deseas modificar o escalar la aplicación, este documento será tu mapa principal.

---

## 1. Visión General del Sistema (El Stack)

La aplicación está construida sobre una arquitectura **Cliente-Servidor (Frontend - Backend)** muy moderna y separada (desacoplada).

- **Backend (El Cerebro y la Base de Datos)**: Construido con **Python** usando el framework **Django 6.0** y **Django REST Framework (DRF)**. Su propósito es manejar la base de datos, la seguridad, las validaciones y exponer los datos a través de una API REST.
- **Frontend (La Interfaz Visual)**: Construido con **JavaScript/JSX** usando la librería **React 19** y empaquetado con **Vite**. Se encarga de mostrar la información al usuario, manejar la navegación sin recargar la página (Single Page Application) y reaccionar a las interacciones.
- **Comunicación**: Ambos se comunican mediante peticiones HTTP(S) utilizando el formato **JSON**. Para saber "quién es quién", se utiliza un sistema de seguridad llamado **JWT (JSON Web Tokens)**.

---

## 2. El Backend: Django y DRF

El backend se encuentra en la carpeta raíz y está estructurado bajo la filosofía de "Apps" de Django. Cada "App" es un módulo independiente que maneja una parte específica del negocio.

### Estructura de Módulos (Apps)

#### 2.1. `users/` (Usuarios y Autenticación)
- **Propósito**: Gestionar el acceso a la plataforma. Aquí viven los psicólogos/evaluadores.
- **Archivos clave**:
  - `models.py`: Define el modelo `Psicologo` (si hay una extensión del usuario de Django).
  - `views.py`: Contiene `RegistrationView` (para registrar) y `CustomTokenObtainPairView` (para el login).
  - `urls.py`: Expone los endpoints `/api/users/login/`, `/api/users/register/`, `/api/users/me/`.

#### 2.2. `patients/` (Gestión de Pacientes)
- **Propósito**: Manejar el expediente de los niños evaluados.
- **Archivos clave**:
  - `models.py`: Define el modelo `Paciente` (relacionado con el psicólogo creador). Contiene propiedades útiles como `edad_meses` y `edad_formateada`, ya que la edad exacta en meses es crucial para las pruebas DAYC-2.
  - `views.py`: Expone el CRUD (Crear, Leer, Actualizar, Eliminar) de pacientes.

#### 2.3. `evaluations/` (El núcleo médico / Pruebas)
- **Propósito**: Manejar la lógica central de la prueba DAYC-2.
- **Archivos clave**:
  - `models.py`: 
    - `DominioDAYC`: Los 5 ejes de la prueba (Cognitivo, Lenguaje, Motricidad, Social, Adaptativo).
    - `Evaluacion`: Cabecera que une a un paciente con la prueba que está rindiendo en una fecha y estado (INICIADA, COMPLETADA).
    - `ResultadoDominio`: Guarda los puntajes finales de un paciente en un dominio específico (puntaje directo, percentil, edad equivalente).
    - `ItemRespuesta`: Guarda las respuestas individuales (pregunta a pregunta) de la prueba.
  - `views.py`: Contiene lógica pesada, como el inicio de la prueba (`iniciarEvaluacion`), guardado masivo de respuestas (`enviarRespuestasBatch`) y el cálculo de resultados (`calcularResultados`).

#### 2.4. `reports/` (Generación de Reportes)
- **Propósito**: Crear documentos descargables, típicamente en PDF.
- **Tecnología**: Usa la librería `reportlab` de Python.
- **Dónde buscar**: Si necesitas cambiar cómo se ve el PDF que se imprime al final de una evaluación, debes buscar en las vistas o utilidades de esta carpeta.

#### 2.5. `consents/` (Consentimientos)
- **Propósito**: Manejar aprobaciones legales o éticas vinculadas al uso de datos de los pacientes.

### ¿Cómo modificar el Backend?
- **Si quieres añadir un campo a la base de datos (ej. DNI del paciente)**: Ve a `patients/models.py`, añade el campo, luego añade el campo en `patients/serializers.py` y finalmente corre `python manage.py makemigrations` y `python manage.py migrate`.
- **Si quieres cambiar cómo se calcula un puntaje**: Busca en `evaluations/views.py` (o en un archivo utilitario dentro de esa carpeta).

---

## 3. El Frontend: React + Vite

El código del cliente vive en la carpeta `/frontend/`. Está organizado para ser muy modular y fácil de mantener.

### Herramientas principales utilizadas:
- **Tailwind CSS**: Todo el diseño, colores (`bg-primary-600`), márgenes (`mt-4`) y sombras se hacen mediante clases directamente en los componentes. (Revisa `tailwind.config.js` e `index.css`).
- **Zustand**: Gestor de estado global (ubicado en `src/stores/`). Permite que el token del usuario y sus datos vivan en toda la aplicación sin tener que pasarlos de componente en componente.
- **React Router**: Permite navegar entre `/pacientes`, `/dashboard`, etc., sin recargar el navegador (`src/App.jsx`).
- **Axios**: Librería que envía las peticiones de red al backend.

### Estructura de Módulos (Carpetas dentro de `src/`)

#### 3.1. `api/` (El puente de comunicación)
Aquí ocurre la magia de la conexión.
- **`client.js`**: Este archivo es CRÍTICO. 
  - Crea una instancia de `Axios`.
  - Contiene **Interceptores**: Cada vez que el frontend quiere pedir algo al backend, este interceptor "atrapa" la petición y le **inyecta el Token JWT** automáticamente (`Authorization: Bearer <token>`). Si el token expira o es inválido (Error 401), el interceptor cierra sesión y envía al usuario al `/login`.
  - **Funciones exportadas**: Todas las funciones como `login()`, `register()`, `getPacientes()`, `iniciarEvaluacion()`. Si añades una ruta en Django, debes crear su función homóloga aquí para que React pueda llamarla.

#### 3.2. `stores/` (El Estado Global)
- **`authStore.js`**: Usa Zustand. Define acciones como `login`, `register` y `logout`. Además, persiste la sesión en el `localStorage` (memoria del navegador) para que si el usuario refresca la página, siga dentro.

#### 3.3. `pages/` (Las Vistas Completas)
Cada archivo aquí es una pantalla completa a la que accedes por una URL.
- **`LoginPage.jsx`**: Pantalla de acceso y registro (recientemente mejorada con un diseño premium dividido en dos paneles).
- **`DashboardPage.jsx`**: Panel principal que ve el psicólogo al entrar.
- **`PacientesPage.jsx` / `NuevoPacientePage.jsx`**: Vistas de la tabla de pacientes y formulario de creación.
- **`TestDominioPage.jsx`**: Posiblemente la vista más compleja; aquí es donde el usuario va marcando las respuestas de la evaluación ítem por ítem.
- **`ResultadosPage.jsx` / `ResumenPage.jsx`**: Muestran las gráficas (usando `Recharts`) y resúmenes al finalizar la prueba.

#### 3.4. `components/` (Las Piezas de Lego)
Botones, cuadros de texto, modales, tarjetas. Están diseñados para usarse múltiples veces en diferentes "Pages". Si quieres cambiar la forma en que todos los botones de la plataforma se ven, vas a `src/components/Button.jsx`.

---

## 4. El Flujo Completo: ¿Cómo se conectan el Frontend y Backend?

Para que el propósito quede 100% claro, analicemos el camino que recorre la información usando de ejemplo el **Login y Consulta de Pacientes**:

1. **Interacción (Frontend)**: El usuario entra a `http://localhost:5173/login`, ingresa su usuario y contraseña y da clic a "Ingresar".
2. **Llamada a la API (Frontend)**: El componente `LoginPage.jsx` dispara la función `login(username, password)` del `authStore.js`. Esta a su vez llama a `api.post('/users/login/')` definido en `api/client.js`.
3. **Recepción (Backend)**: La petición HTTP viaja por la red (en este caso tu `localhost`) y llega a Django (`manage.py runserver`). Django revisa su archivo `urls.py` principal y dice: "Ah, la ruta `/api/users/login/` corresponde a la app `users`".
4. **Procesamiento (Backend)**: El código llega a `CustomTokenObtainPairView` en `users/views.py`. Django REST Framework toma la contraseña, verifica si el usuario existe en SQLite (`db.sqlite3`), y si es válido, **firma digitalmente un Token JWT**.
5. **Respuesta (Backend a Frontend)**: Django devuelve un JSON que contiene `{"token": "eyJhb...", "user": {"username": "gonzalo"...}}`.
6. **Guardado del Estado (Frontend)**: La promesa en `client.js` se resuelve. El `authStore.js` toma ese Token y lo guarda en el `localStorage`. Luego avisa a React: "El usuario ya está autenticado".
7. **Navegación (Frontend)**: El `React Router` (`App.jsx`) detecta el cambio, quita el `LoginPage` de la pantalla y dibuja el `DashboardPage`.
8. **Consultar Datos (Frontend)**: Al cargar el Dashboard, un componente hace un llamado a `getPacientes()`. La petición sale hacia Django, pero esta vez, el interceptor de Axios (en `client.js`) le pega el Token en las cabeceras.
9. **Validación de Seguridad (Backend)**: Django recibe la petición en `patients/views.py`, ve el token, valida que nadie lo haya manipulado, y dice: "Es un usuario válido, le devolveré sus pacientes".
10. **Renderizado (Frontend)**: React recibe la lista de pacientes y usa `Tailwind` para dibujar una tabla hermosa en la pantalla.

---

## 5. Mini Guía Rápida para Modificar (Cheatsheet)

- **Quiero cambiar el color primario de toda la aplicación**: 
  Ve a `/frontend/src/index.css` o `/frontend/tailwind.config.js` y busca la paleta de colores `--primary`.
- **Quiero añadir una nueva gráfica de resultados**: 
  Añade/Modifica un componente en `/frontend/src/pages/ResultadosPage.jsx` usando componentes de la librería *Recharts*.
- **La API da un error de que falta un dato al guardar un paciente**: 
  1. Verifica qué envía React: `/frontend/src/api/client.js` -> `createPaciente(data)`.
  2. Verifica qué pide Django: `/patients/serializers.py` (el Serializer define qué campos son obligatorios).
- **Quiero ver todos los endpoints que existen en el servidor**: 
  Navega por los archivos `urls.py` de cada carpeta en el backend, o entra en tu navegador a `http://localhost:8000/api/` (si el DRF browsable API está activo).
- **Quiero hacer que un campo en la base de datos ya no sea obligatorio**:
  Ve a `patients/models.py`, busca el campo y ponle `blank=True, null=True`. Guarda el archivo, abre tu terminal y ejecuta `python manage.py makemigrations` y luego `python manage.py migrate`.

---

Esta guía condensa la forma en que está construida la **Plataforma DAYC-2**. Si alguna vez sientes dudas, recuerda la regla de oro de esta arquitectura: **El Backend maneja las reglas y guarda la información; El Frontend la dibuja y pide permiso.**
