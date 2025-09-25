# Project Athena - Фитнес-экосистема

Архитектурное решение для создания социальной фитнес-платформы транснациональной компании спортивных товаров.

## Структура проекта

### Этап 1: Утверждение концепта архитектурного решения
- [Бизнес-цели](01-concept-approval/01-business-goals.md)
- [Функциональные требования](01-concept-approval/02-functional-requirements.md)
- [Анализ стейкхолдеров](01-concept-approval/03-stakeholders-analysis.md)
- [Концептуальная архитектура](01-concept-approval/04-conceptual-architecture.md)
- [Риски реализации](01-concept-approval/05-implementation-risks.md)
- [План разработки](01-concept-approval/06-development-plan.md)
- [Критические сценарии](01-concept-approval/07-critical-use-cases.md)
- [Атрибуты качества](01-concept-approval/08-quality-attributes.md)
- [Нефункциональные требования](01-concept-approval/09-non-functional-requirements.md)
- [Архитектурные опции](01-concept-approval/10-architectural-options.md)

### Этап 2: Защита плана реализации
- [ADR](02-implementation-defense/11-adr-list.md)
- [Сценарии использования](02-implementation-defense/12-usage-scenarios.md)
- [Базовая архитектура](02-implementation-defense/13-base-architecture.md)
- [Ключевые представления](02-implementation-defense/14-key-views/)
  - [Функциональное](02-implementation-defense/14-key-views/14a-functional-view.md)
  - [Информационное](02-implementation-defense/14-key-views/14b-information-view.md)
  - [Многозадачность](02-implementation-defense/14-key-views/14c-concurrency-view.md)
  - [Инфраструктурное](02-implementation-defense/14-key-views/14d-infrastructure-view.md)
  - [Безопасность](02-implementation-defense/14-key-views/14e-security-view.md)

### Этап 3: Финальное утверждение
- [Риски архитектуры](03-final-approval/15-architecture-risks-compromises.md)
- [Анализ TCO](03-final-approval/16-tco-analysis.md)
- [Презентация](03-final-approval/presentation/)

## Краткое описание

Project Athena - это социальная фитнес-платформа, предназначенная для создания экосистемы вокруг продуктов компании. Платформа сочетает в себе социальные взаимодействия, отслеживание тренировок, геймификацию и прямую интеграцию с e-commerce системой компании.

**Ключевые технологии:**
- Микросервисная архитектура
- Event-Driven Architecture
- Multi-cloud стратегия (AWS + GCP)
- Kubernetes для оркестрации
- GraphQL API для клиентов