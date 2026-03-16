---


---

<h1 id="evaluación-de-perfil-técnico-—-juan-borré-dev-.net-backend-senior">Evaluación de Perfil Técnico — Juan Borré (Dev .NET Backend Senior)</h1>
<blockquote>
<p>Fecha de evaluación: Marzo 2026<br>
Analista: GitHub Copilot<br>
Fuentes: CV adjunto + 5 repositorios NuGet públicos (<code>juanborre87</code>)</p>
</blockquote>
<hr>
<h2 id="resumen-ejecutivo">1. Resumen Ejecutivo</h2>
<p>Juan Borré demuestra un nivel <strong>Senior sólido</strong> en .NET backend. Tiene capacidad comprobada de diseñar y publicar librerías de infraestructura reutilizables (NuGet privados), entiende Clean Architecture y CQRS desde las capas mismas, y trabaja en el stack exacto del proyecto (net8.0, Dapper, FluentValidation, Azure Pipelines). Es un candidato apto para integrarse al sprint en curso, con una curva de adaptación baja a los patrones específicos del proyecto.</p>
<hr>
<h2 id="análisis-de-repositorios">2. Análisis de Repositorios</h2>
<h3 id="solidaria.mediatr-—-reemplazo-propio-de-mediatr">2.1 <code>Solidaria.MediatR</code> — Reemplazo propio de MediatR</h3>

<table>
<thead>
<tr>
<th>Ítem</th>
<th>Detalle</th>
</tr>
</thead>
<tbody>
<tr>
<td>Motivación</td>
<td>Evitar costo de licencia de MediatR 12+</td>
</tr>
<tr>
<td>Target</td>
<td>net8.0 · FluentValidation 12.0</td>
</tr>
<tr>
<td>Patrón</td>
<td><code>IMediatR</code>, <code>IRequest&lt;T&gt;</code>, <code>IRequestHandler&lt;,&gt;</code>, <code>IPipelineBehavior&lt;,&gt;</code></td>
</tr>
<tr>
<td>Dispatch</td>
<td>Via <code>IServiceScopeFactory</code> + genéricos con <code>dynamic</code></td>
</tr>
<tr>
<td>Pipeline</td>
<td>FluentValidation behavior incluido, composición en cadena (Reverse)</td>
</tr>
<tr>
<td>CI/CD</td>
<td>Azure Pipelines → PackAndPublishNuget template</td>
</tr>
</tbody>
</table><p><strong>Observación:</strong> El uso de <code>dynamic</code> + <code>MakeGenericMethod</code> en el overload <code>Send(object)</code> introduce overhead de reflexión. No es un bloqueante, pero en APIs de alto volumen puede ser un punto de optimización. El patrón es funcionalmente equivalente al MediatR que ya usa el proyecto.</p>
<hr>
<h3 id="solidaria.core-—-contratos-cqrs-capa-application">2.2 <code>Solidaria.Core</code> — Contratos CQRS (capa Application)</h3>
<p>Biblioteca de interfaces puras:</p>
<ul>
<li><code>IUnitOfWork</code>: multi-DB, transacciones explícitas (<code>BeginTransactionAsync</code>, <code>CommitAsync</code>, <code>RollbackAsync</code>), acceso a <code>IEFCommandRepository&lt;T&gt;</code>, <code>IEFQueryRepository&lt;T&gt;</code> y <code>IDapperRepository</code></li>
<li><code>IDapperRepository</code>: <code>QueryAsync&lt;T&gt;</code>, <code>QuerySingleAsync&lt;T&gt;</code>, <code>ExecuteAsync</code>, <code>Use(dbChoice)</code>, <code>GetConnection()</code></li>
<li><code>PagedResult&lt;T&gt;</code> + <code>MetaData</code>: paginación genérica</li>
</ul>
<p><strong>Alineación con el proyecto:</strong> el <code>IDapperRepository</code> es conceptualmente idéntico al que usa el proyecto (Dapper + SPs). La diferencia es que aquí el acceso es por <code>dbChoice</code> (string), mientras en el proyecto se accede via SchemaContexts tipados. Adaptable rápidamente.</p>
<hr>
<h3 id="solidaria.cqrs-—-implementación-cqrs-capa-infrastructure">2.3 <code>Solidaria.Cqrs</code> — Implementación CQRS (capa Infrastructure)</h3>

<table>
<thead>
<tr>
<th>Componente</th>
<th>Detalle</th>
</tr>
</thead>
<tbody>
<tr>
<td><code>DapperRepository</code></td>
<td>Implementa <code>IDapperRepository</code>. Soporta transacciones EF + Dapper juntas. Thread-safe via <code>ConcurrentDictionary</code>.</td>
</tr>
<tr>
<td><code>UnitOfWork</code></td>
<td>Gestiona múltiples DbContexts en paralelo. Transacciones por DB o globales (<code>CommitAllAsync</code>).</td>
</tr>
<tr>
<td><code>DbContextProvider</code></td>
<td>Resuelve DbContexts registrados por nombre desde el contenedor DI.</td>
</tr>
<tr>
<td><code>CqrsExtensions</code></td>
<td>Registro DI simplificado.</td>
</tr>
<tr>
<td>Dependencias</td>
<td>Dapper 2.1.66, EF Core Relational 9.0.9, Solidaria.Core 1.0.7</td>
</tr>
</tbody>
</table><p><strong>Nota:</strong> El candidato usa <strong>EF Core</strong> para la gestión de conexiones/transacciones incluso en el repositorio Dapper (hereda <code>DbContext</code>). El proyecto usa <strong>Dapper puro sin EF</strong>. No es un problema de competencia — muestra que conoce ambos mundos — pero requiere que al trabajar en este proyecto abandone EF y use el patrón Dapper + SPs.</p>
<hr>
<h3 id="solidaria.host-—-response-genérica--middleware-capa-presentation">2.4 <code>Solidaria.Host</code> — Response genérica + Middleware (capa Presentation)</h3>
<p>Componentes entregados como NuGet:</p>
<ul>
<li><code>Response&lt;T&gt;</code>: wrapper genérico con <code>Content</code>, <code>StatusCode</code>, <code>IsValid</code>, <code>Notifications</code>, <code>Headers</code></li>
<li><code>Notify</code>: modelo de error/notificación</li>
<li><code>BaseApiController</code>: resuelve <code>IMediatR</code> lazy via <code>HttpContext.RequestServices</code> — evita inyección en constructor en controllers</li>
<li><code>GlobalExceptionMiddleware</code>: mapeo completo de excepciones → HTTP status codes (400/401/403/404/500/501/503/504 + <code>CustomException</code> → 406). Loguea errores y serializa respuesta uniforme</li>
</ul>
<p><strong>Calidad:</strong> Código limpio, usa <code>sealed</code> correctamente, manejo de validación con FluentValidation integrado en el middleware. Patrón similar al que usa <code>Common.Presentation</code> en este proyecto.</p>
<hr>
<h3 id="solidaria.worker.utilities-—-workers-paralelos-adaptativos">2.5 <code>Solidaria.Worker.Utilities</code> — Workers paralelos adaptativos</h3>

<table>
<thead>
<tr>
<th>Componente</th>
<th>Función</th>
</tr>
</thead>
<tbody>
<tr>
<td><code>MultiWorkerHostService&lt;TWorker&gt;</code></td>
<td><code>BackgroundService</code> que levanta N instancias del mismo worker en paralelo</td>
</tr>
<tr>
<td><code>AdaptiveWorkerManager</code></td>
<td>Decide si crear un nuevo worker monitoreando CPU y RAM en tiempo real</td>
</tr>
<tr>
<td><code>SystemResourceMonitor</code></td>
<td>Lee métricas via <code>PerformanceCounter</code> (Windows) + <code>ComputerInfo</code></td>
</tr>
<tr>
<td><code>WorkerFactory&lt;T&gt;</code></td>
<td>Crea instancias de worker usando DI</td>
</tr>
<tr>
<td><code>WorkerExtensions</code></td>
<td>Registro DI con <code>IOptions&lt;WorkerFactorySettings&gt;</code></td>
</tr>
</tbody>
</table><p><strong>Observación notable:</strong> La lógica de control adaptativo (CPU threshold + memory threshold + max instances) es una solución bien pensada para un caso de uso real de workers concurrentes. El uso de <code>CancellationToken</code> y manejo de <code>OperationCanceledException</code> es correcto.</p>
<hr>
<h2 id="alineación-con-el-proyecto-api.core">3. Alineación con el Proyecto Api.Core</h2>

<table>
<thead>
<tr>
<th>Criterio</th>
<th>Estado</th>
<th>Detalle</th>
</tr>
</thead>
<tbody>
<tr>
<td>.NET 8.0</td>
<td>✅ Alineado</td>
<td>Todos los repos son net8.0</td>
</tr>
<tr>
<td>Clean Architecture + CQRS + DDD</td>
<td>✅ Alineado</td>
<td>Lo aplica en sus propias librerías</td>
</tr>
<tr>
<td>Dapper como ORM</td>
<td>✅ Alineado</td>
<td>Dapper 2.1.x presente en Solidaria.Cqrs</td>
</tr>
<tr>
<td>FluentValidation</td>
<td>✅ Alineado</td>
<td>v12.0 integrado en pipeline MediatR</td>
</tr>
<tr>
<td>Stored Procedures exclusivos</td>
<td>⚠️ Por confirmar</td>
<td>No hay evidencia de SPs en los repos. Sus repos usan SQL inline. Necesita adoptar la convención del proyecto con <code>CommandType.StoredProcedure</code></td>
</tr>
<tr>
<td>Sin EF Core</td>
<td>⚠️ Adaptación</td>
<td>En Solidaria.Cqrs usa EF para gestión de transacciones. Debe adaptarse al patrón Dapper puro del proyecto</td>
</tr>
<tr>
<td>Patrón <code>IUnitOfWork</code> + SchemaContexts</td>
<td>⚠️ Adaptación</td>
<td>Su patrón usa <code>dbChoice</code> strings. El proyecto usa schemas tipados (<code>IMaestrosSchemaContext</code>, <code>ICoreSchemaContext</code>). Curva baja</td>
</tr>
<tr>
<td>Azure Pipelines</td>
<td>✅ Alineado</td>
<td>Pipelines configurados en todos los repos</td>
</tr>
<tr>
<td>Tests unitarios</td>
<td>❌ No visible</td>
<td>Ningún repo tiene tests. Punto de riesgo para un proyecto con xUnit + coverlet</td>
</tr>
<tr>
<td>Convenciones de nombrado</td>
<td>⚠️ A validar</td>
<td>Sus repos no siguen el esquema <code>{Entidad}{Accion}Command/Query</code> del proyecto — debe adoptarlo</td>
</tr>
<tr>
<td>StyleCop</td>
<td>❌ No configurado</td>
<td>El proyecto exige StyleCop.Analyzers, no hay evidencia de uso</td>
</tr>
</tbody>
</table><hr>
<h2 id="habilidades-derivadas-del-código">4. Habilidades Derivadas del Código</h2>

<table>
<thead>
<tr>
<th>Competencia</th>
<th>Nivel inferido</th>
</tr>
</thead>
<tbody>
<tr>
<td>Diseño de arquitectura de librerías</td>
<td><strong>Alto</strong> — 5 NuGets publicados con CI/CD, versionado semántico, símbolos (snupkg)</td>
</tr>
<tr>
<td>Conocimiento de DI y ciclo de vida</td>
<td><strong>Alto</strong> — <code>IServiceScopeFactory</code>, <code>ServiceLifetime</code>, scopes por request</td>
</tr>
<tr>
<td>Concurrencia / workers</td>
<td><strong>Alto</strong> — <code>ConcurrentDictionary</code>, adaptive worker management, <code>CancellationToken</code> correcto</td>
</tr>
<tr>
<td>Middleware <a href="http://ASP.NET">ASP.NET</a> Core</td>
<td><strong>Alto</strong> — <code>IMiddleware</code>, mapeo de excepciones, serialización de respuesta</td>
</tr>
<tr>
<td>CQRS / mediador</td>
<td><strong>Alto</strong> — implementó su propia versión funcional</td>
</tr>
<tr>
<td>Dapper</td>
<td><strong>Medio-Alto</strong> — usa la API correctamente aunque lo combina con EF</td>
</tr>
<tr>
<td>Testing</td>
<td><strong>Bajo</strong> — no hay tests en ningún repo</td>
</tr>
<tr>
<td>Documentación técnica</td>
<td><strong>Bajo</strong> — READMEs son placeholders (<code>"Clean Architecture"</code>)</td>
</tr>
</tbody>
</table><hr>
<h2 id="señales-positivas-para-el-sprint">5. Señales Positivas para el Sprint</h2>
<ul>
<li>Capacidad de <strong>producir código de infraestructura rápidamente</strong> — los 5 NuGets son funcionales y publicados, lo que indica velocidad y criterio arquitectónico</li>
<li>Motivación intrínseca para <strong>resolver problemas de dependencias de terceros</strong> (reemplazar MediatR pagado) — fit cultural con un equipo que controla sus propios paquetes</li>
<li><strong>Código en español</strong> con logging en español — alineado con el estilo del proyecto</li>
<li>Entiende el <strong>patrón Request/Handler</strong> de la misma forma que el proyecto lo usa — puede leer y extender comandos/queries existentes desde el día 1</li>
</ul>
<hr>
<h2 id="riesgos-y-recomendaciones">6. Riesgos y Recomendaciones</h2>

<table>
<thead>
<tr>
<th>Riesgo</th>
<th>Mitigación sugerida</th>
</tr>
</thead>
<tbody>
<tr>
<td>Sin tests visibles</td>
<td>Validar en entrevista si trabajó con xUnit/Moq en otros proyectos. El proyecto exige cobertura</td>
</tr>
<tr>
<td><code>dynamic</code> en MediatR propio no aplica aquí</td>
<td>No es un problema — el proyecto usa el Common.Application con MediatR estándar. Su implementación es para otro producto</td>
</tr>
<tr>
<td>Adaptación a SPs exclusivos</td>
<td>Sesión de onboarding enfocada en el patrón Dapper→SP del proyecto (1 día es suficiente)</td>
</tr>
<tr>
<td>EF Core vs. Dapper puro</td>
<td>Debe no introducir EF en el proyecto. Importante aclarar en el onboarding</td>
</tr>
<tr>
<td>Documentación mínima</td>
<td>Establecer expectativa clara de comentarios XML en interfaces y commands nuevos</td>
</tr>
</tbody>
</table><hr>
<h2 id="veredicto">7. Veredicto</h2>

<table>
<thead>
<tr>
<th>Dimensión</th>
<th>Puntuación</th>
</tr>
</thead>
<tbody>
<tr>
<td>Solidez técnica .NET</td>
<td>⭐⭐⭐⭐⭐</td>
</tr>
<tr>
<td>Alineación arquitectónica</td>
<td>⭐⭐⭐⭐☆</td>
</tr>
<tr>
<td>Velocidad de entrega</td>
<td>⭐⭐⭐⭐⭐</td>
</tr>
<tr>
<td>Testing</td>
<td>⭐⭐☆☆☆</td>
</tr>
<tr>
<td>Documentación</td>
<td>⭐⭐☆☆☆</td>
</tr>
</tbody>
</table><p><strong>Recomendación: APTO para incorporar al equipo como Senior .NET Backend.</strong></p>
<p>El perfil técnico está por encima del promedio del mercado en arquitectura de software .NET. La brecha más relevante no es de conocimiento sino de hábitos: testing y documentación. Dado que el equipo tiene convenciones establecidas (xUnit, StyleCop, CONTEXTO_PROYECTO.md), con un onboarding bien estructurado de 2-3 días el candidato puede estar contribuyendo al sprint en curso desde la primera semana.</p>
<hr>
<p><em>Reporte generado con análisis de código fuente de repositorios públicos + CV. No reemplaza la entrevista técnica.</em></p>

