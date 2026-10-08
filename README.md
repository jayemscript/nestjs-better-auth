# NestJS + Better Auth Framework

An authentication and identity backend service built with **NestJS**, **Better Auth**, **PostgreSQL**, and **TypeORM**.

Designed as a reusable authentication service that can be integrated into different application architectures through **REST APIs / HTTP**.

## Use Cases

* **Monolithic Applications**
  Use the service directly through REST APIs or HTTP as the application's authentication and identity layer.

* **Multi-Tenant / SaaS Platforms**
  Centralize authentication across multiple applications, services, or tenants with support for **SSO**.

* **SaaS Applications**
  Provide a dedicated authentication backend that can be reused across multiple client applications.

* **Internal Applications**
  Includes a **SuperAdmin seeder** that can be executed with a single command to generate the initial SuperAdmin account and credentials.

## Features

* Email / Username & Password
* Google OAuth
* Two-Factor Authentication (2FA)
* Magic Links
* Email OTP
* Passkeys
* Single Sign-On (SSO)
* Account Management
* Session Management

## Tech Stack

* **NestJS**
* **Better Auth**
* **PostgreSQL**
* **TypeORM**
* **REST API / HTTP**
