# 🧠 Sistema de Análisis de Feedback — UrbanThread

**Proyecto Final · AI Automation · Coderhouse**
**Estudiante:** Valeria Villegas · **Fecha:** 23 de marzo de 2026

---

## 📌 Descripción

Sistema de análisis automatizado de feedback de clientes para **UrbanThread**, una tienda de indumentaria online. El workflow procesa comentarios en tiempo real, los clasifica según su nivel de riesgo reputacional mediante inteligencia artificial y genera respuestas e informes accionables **sin intervención humana**.

> ⚡ Tiempo de respuesta ante una crisis: **menos de 60 segundos** desde que el cliente escribe su comentario.

---

## 🎯 ¿Qué hace este sistema?

Cada vez que un cliente deja un comentario en Google Sheets, el workflow:

1. **Valida** que el comentario sea nuevo y no esté vacío
2. **Clasifica** el sentimiento con IA: `CRISIS` / `NEGATIVO` / `POSITIVO` / `ERROR`
3. **Actúa** de forma diferente según la clasificación:
   - 🚨 **CRISIS** → Genera análisis urgente + envía email de alerta al equipo
   - ⚠️ **NEGATIVO** → Genera análisis detallado + envía email con acciones sugeridas
   - ✅ **POSITIVO** → Extrae insights, señal NPS y posible testimonio
   - ❌ **ERROR** → Registra el fallo y notifica al equipo técnico
4. **Registra** todo en hojas de Google Sheets para métricas y seguimiento

---

## 🗺️ Diagrama del Workflow

![Diagrama del workflow](Workflow_-_Sistema_de_Análisis_de_Feedback.png)

### Workflow completo en n8n
![Captura de workflow](https://github.com/user-attachments/assets/1ad72955-ef99-4f6a-b855-1ea99cb17811) 

---

## 🏗️ Arquitectura

```
Google Sheets Trigger
        │
        ▼
Code in JavaScript ──── (vacío o ya procesado → descarta)
        │
        ├──► Sheets — Marcar Procesado (paralelo)
        │
        ▼
LLM 1 — Clasificador de Severidad (Groq llama-3.3-70b)
        │
        ▼
Router — Severidad
   │         │         │         │
   ▼         ▼         ▼         ▼
ERROR    NEGATIVO   POSITIVO   CRISIS
   │         │         │         │
   ▼         ▼         ▼         │
Handler   LLM 2     LLM 3      LLM 2
Errores   Crisis    Positivos  Crisis
   │         │         │         │
   ▼         ▼         ▼         ▼
Log +    Gmail +   Log +     Gmail +
Gmail    Log       Log NPS   Log
             │         │         │
             └────┬─────┘         │
                  ▼               ▼
            Sheets Métricas   Sheets Métricas
```

---

## 🤖 Nodos LLM

| # | Nodo | Modelo | Temperatura | Rol |
|---|------|--------|-------------|-----|
| 1 | Clasificador de Severidad | `llama-3.3-70b-versatile` | 0.1 | Clasifica el comentario y devuelve JSON estructurado |
| 2 | Analista de Crisis | `llama-3.3-70b-versatile` | 0.3 | Genera análisis completo para CRISIS y NEGATIVO |
| 3 | Destacador de Positivos | `llama-3.3-70b-versatile` | 0.2 | Extrae insights y señal NPS de comentarios positivos |

### Ejemplo de salida estructurada — LLM 1

```json
{
  "severity": "CRISIS",
  "score": 0.96,
  "topic": "fraude cobro",
  "urgency": "ALTA",
  "category": "precio"
}
```

---

## 📂 Estructura del repositorio

```
Proyecto_final_coderhouse/
│
├── workflow.json                          # Workflow exportado de n8n
├── Documentacion_Tecnica_UrbanThread.docx # Documentación técnica completa
├── prompts.txt                            # Prompts de los 3 nodos LLM
├── Workflow_-_Sistema_de_Análisis_de_Feedback.png  # Diagrama del flujo
├── evidencia/                             # Capturas de pruebas
│   ├── prueba_crisis.png
│   ├── prueba_negativo.png
│   ├── prueba_positivo.png
│   └── prueba_error.png
└── README.md                              # Este archivo
```

---

## 🚀 Cómo importar y ejecutar

### Requisitos previos

- Instancia de **n8n** activa (cloud o self-hosted)
- Cuenta de **Google** (para Sheets y Gmail)
- Cuenta en **Groq** (gratuita) para el modelo LLM

### Paso a paso

**1. Importar el workflow**
```
n8n → menú superior → Import → From file → seleccionar workflow.json
```

**2. Configurar credenciales** (Settings → Credentials → Add Credential)

| Credencial | Tipo en n8n | Cómo obtenerla |
|-----------|-------------|----------------|
| Google Sheets (trigger) | Google Sheets Trigger OAuth2 | Google Cloud Console → APIs → OAuth2 |
| Google Sheets (lectura/escritura) | Google Sheets OAuth2 API | Misma app OAuth2 |
| Gmail | Gmail OAuth2 API | Misma app OAuth2, scope de envío |
| Groq | Groq API | [console.groq.com](https://console.groq.com) → API Keys |

> ⚠️ Las credenciales **no están incluidas** en el `workflow.json`. Cada usuario debe configurar las propias.

**3. Preparar Google Sheets**

Crear un spreadsheet con estas 5 hojas exactamente:

| Hoja | Propósito |
|------|-----------|
| `Feedback` | Hoja de entrada — trigger del workflow |
| `Crisis_Negativo` | Log de análisis de casos críticos |
| `Positivos` | Log de insights positivos |
| `Errores` | Log de fallos técnicos |
| `Metricas` | Métricas diarias para dashboard |

Columnas de la hoja **Feedback**:
```
fecha_hora | id_cliente | correo | comentario | canal | producto | procesado
```

**4. Conectar el Sheet al workflow**
- Abrir el nodo `Google Sheets Trigger` → seleccionar el documento y la hoja `Feedback`
- Repetir para cada nodo de Google Sheets del workflow

**5. Activar**
```
Toggle superior derecho del workflow → ON
```

---

## 🧪 Casos de prueba

Agregá estas filas en la hoja `Feedback` (campo `procesado` debe estar **vacío**):

### Prueba 1 — CRISIS
```
fecha_hora:  2026-03-22 10:00
id_cliente:  EVAL-001
correo:      evaluador@test.com
comentario:  Me estafaron, me cobraron el doble y nadie responde. Voy a ir a Defensa del Consumidor y lo voy a viralizar en TikTok.
canal:       web
producto:    zapatillas
```
✅ **Resultado esperado:** Email urgente recibido + fila en `Crisis_Negativo` + fila en `Metricas`

### Prueba 2 — POSITIVO
```
fecha_hora:  2026-03-22 15:00
id_cliente:  EVAL-002
correo:      evaluador@test.com
comentario:  Todo llegó perfecto, muy rápido y bien empaquetado. Lo recomiendo 100% a todos mis amigos.
canal:       instagram
producto:    remera
```
✅ **Resultado esperado:** Fila en `Positivos` con NPS PROMOTOR + fila en `Metricas`

### Prueba 3 — NEGATIVO
```
fecha_hora:  2026-03-22 16:00
id_cliente:  EVAL-003
correo:      evaluador@test.com
comentario:  El producto llegó roto y el servicio al cliente no me dio solución después de 4 llamadas.
canal:       google
producto:    mochila
```
✅ **Resultado esperado:** Email de análisis recibido + fila en `Crisis_Negativo` + fila en `Metricas`

---

## 🎁 Mejoras opcionales implementadas

- ✅ **3 nodos LLM** con roles diferenciados (supera el mínimo de 2)
- ✅ **Métricas diarias** en hoja separada para dashboard
- ✅ **Draft para redes sociales** generado automáticamente (hasta 280 caracteres)
- ✅ **Análisis por categoría** además del sentimiento
- ✅ **Handler de errores** con notificación al equipo y log estructurado
- ✅ **Mecanismo anti-duplicados** con idempotencia garantizada

---

## 📊 Cumplimiento de la rúbrica

| Criterio | Peso | Estado |
|----------|------|--------|
| Funcionalidad básica (trigger, LLMs, condicional, notificación) | 40% | ✅ Completo |
| Calidad del LLM clasificador | 20% | ✅ Completo |
| Calidad del LLM generador | 15% | ✅ Completo |
| Documentación y reproducibilidad | 15% | ✅ Completo |
| Robustez y manejo de errores | 10% | ✅ Completo |

---

## 🔒 Seguridad

- Las credenciales **no se incluyen** en ningún archivo del repositorio
- Google APIs usan **OAuth2** con scopes mínimos necesarios
- La API Key de Groq se almacena cifrada en el sistema de credenciales de n8n
- El mecanismo anti-duplicados protege contra reprocesamiento accidental

---

## 📎 Evidencia de pruebas

🎥 [Ver video de demostración en Google Drive](https://drive.google.com/file/d/19GbQzXX6yxTKNKMdpNEFiUwzUSwuJYYw/view?usp=sharing)

> El video muestra la ejecución en tiempo real de los 4 casos de prueba, los emails recibidos y el registro en Google Sheets.

---

## 📄 Documentación

La documentación técnica completa se encuentra en el archivo `Documentacion_Tecnica_UrbanThread.docx`, que incluye:

- Arquitectura detallada del workflow
- Prompts completos de los 3 nodos LLM
- Estructura de datos de todas las hojas
- Guía de configuración para evaluadores
- Limitaciones y sesgos del sistema

---

*Proyecto Final — AI Automation · Coderhouse · Valeria Villegas · 2026*

