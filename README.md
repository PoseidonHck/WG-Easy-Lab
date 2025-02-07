# Implementación de VPN con WireGuard y wg-easy

## 📌 Descripción
Este repositorio contiene la documentación y configuración utilizada para la implementación de un servidor VPN utilizando **WireGuard** y la herramienta **wg-easy**. El objetivo es simular el acceso remoto a una LAN empresarial, permitiendo a los clientes conectarse a recursos internos de manera segura a través de la VPN.

## 🚀 Herramientas Utilizadas
- **WireGuard** - VPN moderna, rápida y segura.
- **wg-easy** - Aplicación web para administrar WireGuard de forma sencilla.
- **Docker & Docker-Compose** - Para desplegar el servidor VPN en contenedores.
- **iptables** - Para la configuración del enrutamiento y NAT.
- **Apache2** - Para simular un recurso interno accesible solo a través de la VPN.
- **VMware Workstation** - Para la creación y gestión de máquinas virtuales.
- **Ubuntu Server 24.01 & Ubuntu Desktop** - Sistemas operativos utilizados en las máquinas virtuales.

## 📄 Contenido
- **Configuración del servidor VPN con wg-easy**
- **Creación de clientes VPN**
- **Configuración de reglas NAT y firewall**
- **Simulación de acceso a recursos internos**

## 📢 Créditos y Recursos
Este proyecto ha sido posible gracias a la información y herramientas proporcionadas por:
- 📌 [wg-easy en GitHub](https://github.com/wg-easy/wg-easy)
- 🎥 [Canal de YouTube DriveMeca](https://www.youtube.com/@DriveMeca)
