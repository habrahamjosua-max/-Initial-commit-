# Rifas Toluca — Full Stack Raffle System (Pulsar RR200 2025)

Sistema completo para administrar rifas en línea, construido con **Node.js + PostgreSQL + React**.  
Diseñado para vender hasta **100,000 boletos**, asignarlos automáticamente, **enviar notificaciones por WhatsApp** y **realizar el sorteo final desde un panel de administrador**.

---

## 🚀 Características principales

| Función | Descripción |
|---------|-------------|
| 🎫 Generación automática de boletos | 100,000 boletos precargados en base de datos |
| 🛒 Compra de boletos | El usuario elige cantidad y el sistema asigna números al azar |
| 📲 Notificación por WhatsApp | Se envía confirmación con los boletos comprados vía Twilio |
| 📊 Panel de administrador | Ver boletos vendidos, exportar datos, ejecutar sorteo |
| 🔄 Integración opcional de pagos | Stripe / MercadoPago listos para conectar |
| 🧾 Logs y auditoría | Historial de compras y usuarios registrados |

---

## 🏗️ Stack Tecnológico

| Capa | Tecnología |
|------|------------|
| Backend API | Node.js + Express |
| Base de datos | PostgreSQL |
| Frontend | React (con Vite) |
| Notificaciones | Twilio WhatsApp API |
| Deploy recomendado | Render / Railway (Backend), Vercel / Cloudflare Pages (Frontend) |

---

## 📂 Estructura del Proyecto

