# DeclaraFácil

SaaS de declaraciones tributarias para el **SRI de Ecuador**, pensado como producto multiusuario licenciable a contadores — no como script de un solo contribuyente.

> **Sobre este repositorio:** es una vitrina técnica. Documenta las decisiones de arquitectura y los hallazgos del dominio. El código fuente es privado por ser un producto comercial.

---

## Qué resuelve

Una persona natural no obligada a llevar contabilidad que factura servicios profesionales recibe retenciones de IVA y de renta, y debe presentar cada mes el **Formulario 104** y el **ATS**, más el **Formulario 102** anual.

Hacerlo a mano significa abrir decenas de comprobantes XML, clasificar cada gasto, sumar bases por tarifa y transcribir casillero por casillero. Un usuario real maneja ~50 facturas por período.

DeclaraFácil toma los comprobantes electrónicos, los clasifica, y produce los valores de cada casillero y el XML del ATS listo para subir.

---

## Tres hallazgos del dominio que cambian el resultado

Estos no son detalles de implementación: son la diferencia entre una declaración correcta y una con errores que el SRI rechaza.

### 1. Las retenciones del formato 2.0.0 viven en otro nodo

El comprobante de retención en **formato 2.0.0** expone los valores bajo:

```xml
<docsSustento>
  <retenciones>…</retenciones>
</docsSustento>
```

…y **no** bajo el `<impuestos><impuesto>` del formato anterior.

Un parser que lee solo el nodo antiguo reporta retenciones grandes de IVA y de renta como **$0**. Silenciosamente. Sin error.

### 2. El IVA se calcula `base × tarifa`, no sumando por factura

El método correcto —el que usa el propio SRI— es tomar la **base imponible total** y multiplicarla por la tarifa:

```
IVA = base_total × tarifa
```

No sumar el IVA factura por factura. La diferencia aparece como descuadre de centavos contra la declaración real: en un caso concreto, **$674,25 frente a $674,21**.

### 3. El ATS hay que validarlo contra el ejemplo oficial

La estructura del Anexo Transaccional Simplificado se verificó **nodo por nodo** contra el archivo de ejemplo publicado por el SRI, no contra la documentación.

---

## Arquitectura

```
Comprobantes XML  ──►  Parser  ──►  Clasificación  ──►  Cálculo  ──►  Salidas
(facturas +            (propio)     (reglas + IA)      F104/F102     ATS XML
 retenciones)                                                        casilleros
```

### Clasificación: reglas primero, modelo después

La clasificación de gastos combina **reglas determinísticas por proveedor** con un modelo de lenguaje (Claude) para lo que las reglas no cubren.

El orden importa, y lo aprendí rompiéndolo: hubo un momento en que la presencia de una clave de IA hacía que el sistema **saltara las reglas por completo**, de modo que el modelo sobrescribía en silencio correcciones que el usuario ya había hecho. Arreglar esa precedencia —reglas primero, modelo solo para el resto— eliminó una clase entera de errores repetidos.

### Elegir modelo con datos, no con moda

Se midió **Haiku vs Sonnet** con facturas reales. Gana Sonnet, pero el hallazgo útil fue otro: **el cuello de botella eran las reglas y el prompt, no el modelo**.

---

## Automatización del portal del SRI

El portal tributario no tiene API pública. La descarga de comprobantes se automatiza vía **Chrome DevTools Protocol**, operando una pestaña real del navegador del usuario en lugar de un navegador headless.

Validado con **47 comprobantes reales**.

---

## Licenciamiento offline con Ed25519

El producto se vende a contadores, así que necesita verificar la licencia **sin depender de un servidor**.

- Las licencias se firman con **Ed25519**; la app verifica la firma localmente.
- La clave privada **nunca se distribuye**.
- La prueba gratuita se ancla en el Registro de Windows, de modo que borrar los datos de la aplicación no la reinicia.

---

## Distribución sin certificado de firma de código

Un certificado de firma cuesta cientos de dólares al año. Para vender sin él, el producto se distribuye en dos formatos:

| Formato | Cómo funciona |
|---|---|
| **App de escritorio Electron** | Electron es la ventana; el servidor Next corre bajo Node real |
| **Paquete portable (ZIP, 47 MB)** | Incluye un `node.exe` **ya firmado** por Node.js más la PWA |

El paquete portable conserva la descarga desde el SRI y los datos locales del usuario.

---

## Proxy de IA con cuota por plan

Para que el contador no tenga que conseguir y pagar su propia clave de API, hay un proxy en **Cloudflare Workers**: la licencia firmada actúa como token de autenticación y la cuota se aplica según el plan contratado.

---

## Stack

- **TypeScript** · **Next.js** (App Router) · **Tailwind CSS**
- **SQLite** en modo WAL para los datos locales
- **Electron** para la app de escritorio · **PWA** para el paquete portable
- **Claude API** para clasificación estructurada
- **Cloudflare Workers** para el proxy de IA
- **Chrome DevTools Protocol** para la automatización del portal
- **Ed25519** para la firma de licencias

---

## Dominio cubierto

Comprobantes electrónicos (facturas y retenciones, formatos 1.0.0 y 2.0.0) · Anexo Transaccional Simplificado (ATS) · Formulario 104 (IVA) · Formulario 102 (Impuesto a la Renta) con tabla vigente, impuesto causado y rebaja por gastos personales · mapa de casilleros verificado contra declaraciones reales.

---

*Construido por [José David Villavicencio Canelos](https://github.com/josevillanelos95-crypto) — Magíster en Dirección Financiera, mención Fintech*
