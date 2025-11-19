# Analítica Web — Guía Completa

**Resumen:** Documento de referencia para diseñar, implementar y operar una plataforma de analítica web moderna. Contiene conceptos, plan de medición, fragmentos de implementación, ejemplos de eventos, consultas y prácticas de cumplimiento.

**Badges:** ![Estado](https://img.shields.io/badge/estado-en%20progreso-yellow) ![Licencia](https://img.shields.io/badge/licencia-MIT-blue)

**Tabla de Contenidos**
- **Visión General:**: Descripción y objetivos del proyecto
- **Objetivos de Medición:**: KPIs y métricas principales
- **Plan de Medición:**: Eventos, dimensiones y esquema de datos
- **Implementación:**: Snippets (GA4, GTM, Matomo), DataLayer
- **Modelado y Almacenamiento:**: Ejemplo de esquema y consultas SQL
- **Reportes y Dashboards:**: Ejemplos y plantillas
- **Privacidad y Cumplimiento:**: Consentimiento y retención
- **Operaciones y Troubleshooting:**: Checklist y problemas comunes
- **Glosario y Recursos:**: Términos y lecturas recomendadas

**Visión General**
- **Propósito:**: Proveer una guía práctica para capturar, transformar y reportar datos de interacción web.
- **Audiencia:**: Analistas de datos, ingenieros de datos, product managers y equipos de marketing.
- **Alcance:**: Desde la instrumentación en el front-end hasta consultas y dashboards para negocio.

**Objetivos de Medición (KPIs principales)**
- **Usuarios Activos:**: Usuarios únicos (diarios, semanales, mensuales)
- **Tasa de Conversión:**: Conversiones / sesiones o usuarios según el funnel
- **Tasa de Rebote / Engranche:**: Métricas de interacción por página/usuario
- **Ingresos Atribuidos:**: Ingresos por canal/campaña/producto
- **Duración de Sesión:**: Tiempo promedio y distribución

**Plan de Medición (ejemplo resumido)**
- **Evento:** `page_view` —: Propósito: medir vistas de página. Parámetros: `page_title`, `page_location`, `referrer`, `user_id`.
- **Evento:** `product_view` —: Propósito: medir visualización de producto. Parámetros: `product_id`, `product_name`, `price`, `category`.
- **Evento:** `add_to_cart` —: Propósito: seguimiento de carro. Parámetros: `product_id`, `quantity`, `value`.
- **Evento:** `purchase` —: Propósito: transacción completada. Parámetros: `transaction_id`, `value`, `currency`, `items` (array).

**Esquema sugerido (Data Layer simplificado)**
```json
{
	"event": "page_view",
	"page": {
		"title": "Inicio",
		"location": "https://ejemplo.com/",
		"referrer": "https://google.com"
	},
	"user": {
		"id": "user_123",
		"logged_in": true
	}
}
```

**Implementación — Snippets**

- **Google Analytics 4 (GA4) — gtag.js básico:**
```html
<!-- Global site tag (gtag.js) - Google Analytics -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-XXXXXXXXXX"></script>
<script>
	window.dataLayer = window.dataLayer || [];
	function gtag(){dataLayer.push(arguments);}
	gtag('js', new Date());
	gtag('config', 'G-XXXXXXXXXX', { 'send_page_view': false });
	// Luego disparar page_view manualmente cuando DataLayer esté listo
	gtag('event','page_view', { 'page_title': document.title, 'page_location': location.href });
</script>
```

- **Google Tag Manager (GTM) — contenedor básico:**
```html
<!-- Google Tag Manager -->
<script>(function(w,d,s,l,i){w[l]=w[l]||[];w[l].push({'gtm.start':
new Date().getTime(),event:'gtm.js'});var f=d.getElementsByTagName(s)[0],
j=d.createElement(s),dl=l!='dataLayer'?'&l='+l:'';j.async=true;j.src=
'https://www.googletagmanager.com/gtm.js?id='+i+dl;f.parentNode.insertBefore(j,f);
})(window,document,'script','dataLayer','GTM-XXXX');</script>
<!-- End Google Tag Manager -->
```

- **Matomo (auto-hosted) — tracking básico:**
```html
<script>
	var _paq = window._paq = window._paq || [];
	_paq.push(['trackPageView']);
	_paq.push(['enableLinkTracking']);
	(function() {
		var u="https://matomo.ejemplo.com/";
		_paq.push(['setTrackerUrl', u+'matomo.php']);
		_paq.push(['setSiteId', '1']);
		var d=document, g=d.createElement('script'), s=d.getElementsByTagName('script')[0];
		g.async=true; g.src=u+'matomo.js'; s.parentNode.insertBefore(g,s);
	})();
</script>
```

**Eventos recomendados y nombres estandarizados**
- **Naming:** usar `snake_case` o `kebab-case` consistente; preferible `snake_case`.
- **Categorías:** `engagement`, `ecommerce`, `auth`, `error`.
- **Ejemplo:** `ecommerce_add_to_cart`, `user_signup`, `video_play`.

**Modelado y Almacenamiento (ejemplo SQL)**
- **Tabla eventos (BigQuery / Postgres):**
	- `event_id` (STRING/UUID)
	- `event_name` (STRING)
	- `event_timestamp` (TIMESTAMP)
	- `user_id` (STRING)
	- `session_id` (STRING)
	- `params` (JSON)

**Consulta ejemplo — usuarios únicos por día (BigQuery SQL):**
```sql
SELECT
	DATE(event_timestamp) AS date,
	COUNT(DISTINCT user_id) AS unique_users
FROM `project.dataset.events`
WHERE event_timestamp BETWEEN TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
GROUP BY date
ORDER BY date DESC;
```

**Consulta ejemplo — tasa de conversión por canal:**
```sql
WITH funnel AS (
	SELECT
		user_id,
		MAX(CASE WHEN event_name = 'session_start' THEN 1 ELSE 0 END) AS started,
		MAX(CASE WHEN event_name = 'purchase' THEN 1 ELSE 0 END) AS purchased,
		(SELECT value.string_value FROM UNNEST(params) WHERE key='channel') AS channel
	FROM `project.dataset.events`
	WHERE DATE(event_timestamp) BETWEEN DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY) AND CURRENT_DATE()
	GROUP BY user_id, channel
)
SELECT
	channel,
	SUM(purchased)/SUM(started) AS conversion_rate
FROM funnel
GROUP BY channel
ORDER BY conversion_rate DESC;
```

**Reportes y Dashboards (plantillas)**
- **Dashboard ejecutivo:**: Usuarios activos, sesiones, tasa de conversión, ingresos, tendencia 30d.
- **Dashboard producto:**: Visualizaciones por página, embudos por feature, eventos clave.
- **Dashboard marketing:**: Rendimiento por campaña, coste por adquisición, LTV estimado.

**Privacidad y Cumplimiento**
- **Consentimiento (GDPR/CCPA):**: Implementar un CMP (Consent Management Platform) y respetar el estado de consentimiento para cookies y tracking.
- **Anonimización:**: Nunca almacenar datos personales directos sin enmascaramiento; usar `user_id` pseudonimizado.
- **Retención de datos:**: Definir políticas: datos agregados por 25-26 meses, raw events por 6-12 meses según regulación.

**Operaciones y Troubleshooting**
- **Checklist de implementación:**
	- `dataLayer` presente en todas las páginas.
	- Eventos críticos (purchase, add_to_cart) validados en staging.
	- Variables y parámetros documentados en el plan de medición.
	- Alertas para caídas en eventos principales.
- **Problemas comunes:**
	- Duplicado de page_view: verificar `send_page_view` y GTM triggers.
	- Falta de user_id en eventos: revisar autenticación y push al DataLayer.

**Buenas prácticas**
- **Versionar el plan de medición:** mantenerlo en `docs/measurement_plan.md`.
- **Tests automatizados:** usar test suites que validen presencia/estructura de eventos.
- **Monitoreo:** crear alertas (por ejemplo en Looker/GA/BigQuery) para anomalías.

**Glosario y Recursos**
- **Data Layer:**: Capa JS estructurada que comunica la aplicación con el gestor de etiquetas.
- **CMP:**: Consent Management Platform.
- **GA4:**: Google Analytics 4 — modelo de eventos y esquema de medición.
- **Matomo:**: Alternativa open-source para analítica.

**Contribuidores**
- **Autor:**: `@Jorgerohi2314` 

**Licencia**
- Este documento se publica bajo `MIT`.

---

Si quieres, puedo:
- Generar un `docs/measurement_plan.md` detallado.
- Añadir plantillas JSON para el `dataLayer` por tipo de evento.
- Crear ejemplos de dashboards para Grafana/Looker/Metabase.

Pide cualquiera de estas opciones y lo preparo.