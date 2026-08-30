# 🚀 Semi-Senior (SSR) Software Engineer Roadmap - 2026

Mi hoja de ruta técnica y profesional para consolidar el nivel Semi-Senior (SSR) en ingeniería de software (.NET, React/Next.js, Cloud e IA).

---

## 🎯 Estado de Progreso

- [x] **1. Autonomía y Resolución de Problemas** — 📄 [Ver apuntes](docs/01-autonomia-resolucion-problemas/)
  - [x] Investigación previa estructurada (2-3 alternativas antes de escalar).
  - [x] Depuración avanzada (Breakpoints condicionales, profiling de memory/CPU).
  - [x] Interpretación de logs y rastreo de causas raíz en Stack Traces.

- [x] **2. Calidad de Código y Arquitectura** — 📄 [Ver apuntes](docs/02-calidad-de-codigo-y-arquitectura/)
  - [x] Principios SOLID y Patrones de Diseño (Factory, Strategy, Repository).
  - [x] Estrategia de Testing (TDD/BDD, Unit & Integration Tests).
  - [x] Identificación y refactorización de Deuda Técnica.

- [x] **3. Diseño de APIs y Contratos Técnicos** — 📄 [Ver apuntes](docs/03-diseno-de-apis-y-contratos/)
  - [x] Estándares RESTful semánticos y conceptos de GraphQL.
  - [x] Documentación viva mediante Swagger/OpenAPI y especificaciones claras.

- [ ] **4. Arquitectura Frontend, Rendimiento y Calidad**
  - [ ] Gestión de estado global y de servidor (TanStack Query/SWR): caché, reintentos automáticos, hidratación limpia.
  - [ ] Profiling y rendimiento de cliente: re-renders innecesarios, Web Vitals (LCP, CLS), Lazy Loading, memory leaks.
  - [ ] Código limpio y componentización: Server/Client Components, Custom Hooks, prevención de Prop Drilling.
  - [ ] Estrategia de Testing en UI: Jest/Vitest + React Testing Library, mocking de HTTP con MSW, estados de carga/error.

- [ ] **5. Seguridad y Resiliencia (DevSecOps Básico)**
  - [ ] Prácticas OWASP Top 10 (BOLA, XSS, Sanitización, RBAC).
  - [ ] Gestión estricta de secretos y variables de entorno por ambiente.
  - [ ] Patrones de resiliencia: Retries, Rate Limiting y Circuit Breaker (Polly).

- [ ] **6. Bases de Datos y Rendimiento**
  - [ ] Modelado eficiente (SQL/NoSQL) e indexación.
  - [ ] Optimización de consultas, Planes de Ejecución y solución al problema N+1.

- [ ] **7. Ecosistema Moderno, Cloud y DevOps**
  - [ ] Contenerización con Docker y Docker Compose para entorno local/multi-contenedor.
  - [ ] Entendimiento de Pipelines de CI/CD (GitHub Actions, GitLab CI, Azure DevOps).
  - [ ] IA como copiloto: Refactorización y generación de casos borde con validación humana.

- [ ] **8. Observabilidad y Producción**
  - [ ] Logging estructurado (Serilog + Seq/App Insights).
  - [ ] Monitoreo por métricas (Golden Signals, Grafana) y alertas.

- [ ] **9. Criterio de Negocio, Lean UX y Habilidades Blandas**
  - [ ] Mapeo de reglas de negocio, Criterios de Aceptación (Gherkin) y casos borde.
  - [ ] Estimación precisa por desglose de tareas.
  - [ ] Code Reviews constructivos y mentoría a desarrolladores Juniors.

---

## 🔗 Repositorios Relacionados

- 📂 [dotnet-clean-architecture-starter](https://github.com/Carlou134) - Plantilla base con Serilog, Docker y CQRS.
- 📂 [nextjs-shadcn-template](https://github.com/Carlou134) - Plantilla frontend con Zustand y TanStack Query.
- 📂 [dsa-and-algorithms](https://github.com/Carlou134) - Retos de lógica y optimización de complejidad O(n).