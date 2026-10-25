---
name: legal-mx
description: Lo legal mínimo para lanzar un SaaS en México. Aviso de privacidad conforme a la LFPDPPP de 2025, términos de servicio, cookies con consentimiento real, disclaimer de IA y facturación CFDI 4.0. Úsala cuando se hable de aviso de privacidad, términos, cookies, datos personales, derechos ARCO, LFPDPPP, CFDI, SAT, RFC, facturación o de lanzar un producto sin haber revisado lo legal.
---

# Legal para un SaaS en México

Lo mínimo para operar en regla. **No es asesoría legal ni fiscal.** Los templates
son un punto de partida para revisarlos con un abogado y un contador antes de
publicarlos.

## Qué necesitas antes de lanzar

| Documento | ¿Obligatorio? | Dónde va |
|---|---|---|
| Aviso de privacidad | Sí, por ley (LFPDPPP) | Footer, registro y página propia |
| Términos de servicio | En la práctica, sí | Footer, registro y página propia |
| Política de cookies y banner | Si usas cookies que no son estrictamente necesarias | Banner y página propia |
| Disclaimer de IA | Muy recomendable | Dentro del producto, junto a lo generado |
| Facturación CFDI | Si te la piden tus clientes o lo exige tu régimen | Flujo de compra o portal |

## La ley que aplica hoy

- La LFPDPPP de 2010 fue **abrogada**. La nueva ley se publicó en el DOF el
  **20 de marzo de 2025** y entró en vigor al día siguiente.
- El **INAI se extinguió**. La autoridad para el sector privado es la
  **Secretaría Anticorrupción y Buen Gobierno**. No cites al INAI en avisos nuevos.
- La ley ordenó emitir un reglamento nuevo. Antes de escribir "y su Reglamento",
  confirma en el DOF que ya se publicó; si no, cumple contra el texto de la ley.
- Puntos que cambian tu aviso:
  - Separa **finalidades necesarias** de **voluntarias**; el consentimiento para las
    voluntarias debe ser libre, informado y específico.
  - Di **cuánto tiempo conservas** los datos y qué pasa al terminar ese plazo.
  - Si hay **decisiones automatizadas** (un modelo que evalúa, recomienda o perfila
    sin que intervenga una persona), dilo y explica cómo oponerse.
  - Las multas llegan a cientos de miles de UMA y se pueden duplicar con **datos
    sensibles** (salud, por ejemplo: peso, medidas o condiciones médicas en una app
    de entrenamiento).

### Contenido mínimo del aviso

1. Identidad y domicilio del responsable
2. Datos que recabas, señalando los sensibles
3. Finalidades, separadas en necesarias y voluntarias
4. Mecanismos para manifestar negativa a las voluntarias
5. Medios para limitar el uso o la divulgación
6. Cómo ejercer los derechos ARCO (gratuito)
7. Plazo de conservación
8. Decisiones automatizadas, si las hay
9. Cómo se notifican los cambios al aviso

La ley de 2025 ya no exige de forma expresa listar las transferencias en el aviso,
pero decir con qué proveedores compartes datos sigue siendo buena práctica y evita
problemas si usas servicios fuera de México.

## Template: aviso de privacidad

```markdown
# Aviso de privacidad — [Producto]

**Última actualización:** [fecha]

**Responsable:** [nombre completo o razón social]
**Domicilio:** [domicilio, ciudad, estado, México]
**Contacto para datos personales:** [correo]

## Qué datos recabamos

- **Cuenta:** nombre, correo y contraseña (guardada con hash, nunca en texto plano).
- **Uso del servicio:** la información que capturas para usar [Producto]
  ([describe: experiencia laboral, medidas corporales, documentos, etc.]).
- **Pago:** lo procesan [Stripe / Mercado Pago]; no guardamos números de tarjeta.
- **Técnicos:** dirección IP, navegador, páginas visitadas.
- [Si aplica] **Datos sensibles:** [cuáles]. Los tratamos solo con tu
  consentimiento expreso, que puedes retirar en cualquier momento.

## Para qué los usamos

**Finalidades necesarias** (sin ellas no podemos darte el servicio):
crear y mantener tu cuenta, prestar el servicio contratado, procesar pagos,
darte soporte y cumplir obligaciones legales y fiscales.

**Finalidades voluntarias** (puedes negarte sin perder el servicio):
enviarte novedades y promociones, y generar estadísticas para mejorar el producto.
Para negarte, escribe a [correo] o desactívalo en [ruta de ajustes].

## Proveedores con los que compartimos datos

Para operar usamos proveedores que tratan datos por nuestra cuenta:
- **[Supabase]**: base de datos y autenticación ([región de servidores]).
- **[Vercel]**: hospedaje ([región]).
- **[Stripe / Mercado Pago]**: procesamiento de pagos.
- **[Proveedor de IA]**: procesa el texto que envías para generar tu resultado.
  [Describe su política vigente de retención y de uso para entrenamiento.]

## Decisiones automatizadas

[Producto] usa inteligencia artificial para generar [qué genera] a partir de tus
datos, sin revisión humana. El resultado es orientativo y no sustituye la
opinión de un profesional. Puedes oponerte a este tratamiento escribiendo a [correo].

## Derechos ARCO

Puedes acceder, rectificar, cancelar u oponerte al tratamiento de tus datos, sin
costo. Escribe a [correo] con tu nombre, el derecho que quieres ejercer, los datos
de que se trata y un medio para acreditar tu identidad. Te responderemos dentro del
plazo que marca la ley.

## Cuánto tiempo los conservamos

Mientras tu cuenta esté activa y hasta [X meses] después de cerrarla, salvo que una
obligación legal o fiscal nos pida guardarlos más tiempo. Al vencer el plazo los
borramos de forma segura.

## Cookies

Usamos cookies necesarias para mantener tu sesión y, solo si las aceptas, cookies de
analítica. Detalle en nuestra [Política de cookies](/cookies).

## Cambios a este aviso

Publicaremos cualquier cambio en esta página con su fecha. Si el cambio es
importante, te avisaremos por correo.

## Ley aplicable

Ley Federal de Protección de Datos Personales en Posesión de los Particulares,
publicada en el DOF el 20 de marzo de 2025. Autoridad: Secretaría Anticorrupción y
Buen Gobierno.
```

## Template: términos de servicio

```markdown
# Términos de servicio — [Producto]

**Última actualización:** [fecha]

1. **Aceptación.** Al crear una cuenta o usar [Producto] aceptas estos términos.
2. **El servicio.** [Producto] [qué hace, en una línea]. Se ofrece tal cual.
3. **Cuentas.** Debes ser mayor de 18 años, dar datos verdaderos y cuidar tu acceso.
4. **Uso aceptable.** No puedes usar [Producto] para actividades ilegales, contenido
   engañoso o difamatorio, acceder a cuentas ajenas, saturar o atacar el servicio,
   manipular a los modelos de IA para saltarte sus límites, ni revenderlo sin permiso.
5. **Precios y pagos.** Los precios están en pesos mexicanos e incluyen IVA. Los pagos
   los procesa [Stripe / Mercado Pago].
6. **Reembolsos.** [Elige y escribe una sola regla clara. Ejemplo: los créditos sin
   usar se reembolsan si lo pides dentro de los 7 días siguientes a la compra.]
7. **Contenido generado con IA.** Puede tener errores. Revísalo antes de usarlo. No
   garantizamos resultados concretos. El contenido que generes es tuyo.
8. **Propiedad intelectual.** La marca, el diseño y el código de [Producto] son de
   [titular].
9. **Responsabilidad.** Nuestra responsabilidad máxima se limita a lo que pagaste en
   los últimos 12 meses, salvo lo que la ley no permita limitar.
10. **Cancelación.** Puedes cerrar tu cuenta cuando quieras. Podemos suspender cuentas
    que violen estos términos.
11. **Cambios.** Avisaremos por correo de cambios importantes con [15] días de
    anticipación.
12. **Ley y jurisdicción.** Leyes de los Estados Unidos Mexicanos; tribunales de
    [ciudad], sin perjuicio de los derechos que te da la Ley Federal de Protección al
    Consumidor.
13. **Contacto.** [correo]
```

## Cookies con consentimiento real

Un banner que solo dice "Aceptar" no es consentimiento. Las cookies necesarias
(sesión) no lo requieren; la analítica y la publicidad sí, y **no deben cargarse
antes** de que la persona acepte.

```jsx
import { useEffect, useState } from 'react';

const KEY = 'consent_analytics'; // 'si' | 'no'

function leer() {
  try { return localStorage.getItem(KEY); } catch { return null; }
}

export function BannerCookies({ onAnalytics }) {
  const [valor, setValor] = useState(leer);

  useEffect(() => {
    if (valor === 'si') onAnalytics(); // aquí, y solo aquí, se carga la analítica
  }, [valor, onAnalytics]);

  if (valor) return null;

  const guardar = (v) => {
    try { localStorage.setItem(KEY, v); } catch {}
    setValor(v);
  };

  return (
    <div role="dialog" aria-label="Cookies" className="banner-cookies">
      <p>
        Usamos cookies necesarias para que el sitio funcione y, si aceptas, de
        analítica para mejorarlo. <a href="/cookies">Más información</a>
      </p>
      <button onClick={() => guardar('no')}>Solo necesarias</button>
      <button onClick={() => guardar('si')}>Aceptar analítica</button>
    </div>
  );
}
```

Pon en la página de cookies una tabla con nombre, propósito, duración y tipo
(necesaria o analítica) de cada cookie, y un enlace para cambiar la elección.

## Disclaimer de IA

```markdown
**Generado con inteligencia artificial.** Puede contener errores: revísalo antes
de usarlo. No sustituye la opinión de [un médico / un nutriólogo / un abogado /
un reclutador profesional], y no garantiza [el resultado que la gente espera].
```

Si el producto da recomendaciones de salud o ejercicio, agrega: "Consulta a un
médico antes de empezar un programa de ejercicio o un plan de alimentación,
sobre todo si tienes alguna condición de salud."

## Facturación CFDI 4.0

- **Necesitas:** RFC, régimen fiscal dado de alta, e.firma y Certificado de Sello
  Digital (CSD), y un proveedor que timbre (PAC). Opciones con API para automatizar:
  Facturapi o Facturama, entre otros. Stripe y Mercado Pago cobran, **no** facturan
  por ti.
- **Datos del receptor que pide la versión 4.0:** RFC, nombre o razón social tal como
  aparece en su constancia, régimen fiscal, código postal del domicilio fiscal y
  uso del CFDI (comúnmente `G03` Gastos en general).
- **Ventas a público en general:** si el cliente no pide factura, esas ventas van en
  una factura global periódica con el RFC genérico `XAXX010101000`.
- **Clave de producto o servicio:** búscala en el catálogo `c_ClaveProdServ` del SAT.
  Para software como servicio suele usarse `81112106` (proveedores de servicios de
  aplicación); confírmala con tu contador.
- **Régimen:** una persona física que empieza suele elegir entre actividad
  empresarial o RESICO, según ingresos. Es decisión de tu contador, no del código.
- **Precios:** muéstralos en pesos con IVA incluido; es lo que exige la protección al
  consumidor en México.

## En la app

```jsx
// Registro: aceptación activa, nunca premarcada
<label>
  <input type="checkbox" required />
  Acepto los <a href="/terminos">Términos de servicio</a> y el{' '}
  <a href="/privacidad">Aviso de privacidad</a>
</label>
```

Si vas a usar datos para finalidades voluntarias (correos de promoción), pide ese
consentimiento en una casilla aparte y opcional.

## Checklist antes de lanzar

- [ ] Aviso de privacidad con la ley de 2025, sin mencionar al INAI
- [ ] Finalidades necesarias y voluntarias separadas, con forma de negarse
- [ ] Plazo de conservación y decisiones automatizadas descritos
- [ ] Términos publicados y aceptados con casilla no premarcada
- [ ] Analítica que solo se carga después del consentimiento
- [ ] Disclaimer de IA junto a lo generado (y de salud, si aplica)
- [ ] Precios en MXN con IVA incluido y regla de reembolso clara
- [ ] Correo para derechos ARCO que alguien sí lee
- [ ] Régimen fiscal y forma de facturar definidos con tu contador
