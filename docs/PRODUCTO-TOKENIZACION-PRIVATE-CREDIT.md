# Tokenización – Private Credit (Visión de producto)

Documento de **nivel de producto** para la tokenización de crédito privado en el ecosistema Trustless Work. Se basa en la documentación oficial y en las capacidades implementadas en este repositorio.

**Referencia oficial:** [Tokenization - Private Credit \| Trustless Work Docs](https://docs.trustlesswork.com/trustless-work/oss-dapps/tokenization-private-credit)  
**Vídeo de producto:** [Loom – Tokenization Private Credit](https://www.loom.com/share/08e5ef97f94f42d1b80bebac733a6383)

---

## 1. Visión y contexto

### Qué es

**Tokenization - Private Credit** es el flujo de producto que permite:

- Representar **participación en un escrow** (crédito privado / proyecto financiado) mediante **tokens** on-chain.
- Que los **inversores** compren esos tokens con USDC y que los fondos vayan al **escrow** del proyecto.
- Que los **titulares de tokens** reciban **retornos (ROI)** de forma proporcional a su participación, a través de un **vault**.

Todo ello sobre **Stellar/Soroban** y usando los **escrows de Trustless Work** como fuente de verdad del proyecto (hitos, liberaciones, disputas).

### Problema que aborda

- **Opacidad:** difícil saber en qué está invertido el capital y cómo se ejecuta el proyecto.
- **Liquidez y trazabilidad:** la participación en crédito privado no suele ser portable ni auditable de forma estándar.
- **Distribución de retornos:** repartir beneficios según participación de forma manual es costoso y propenso a errores.

### Propuesta de valor

- **Transparencia:** el escrow y los hitos son visibles; la tokenización refleja un único escrow por proyecto.
- **Automatización:** compra de tokens → fondos al escrow; ejecución del proyecto → retornos al vault → claim por balance de tokens.
- **Alineación con estándares:** tokens compatibles con el ecosistema Stellar (T-REX-aligned), escrows Trustless Work, contratos reutilizables.

---

## 2. Usuarios y roles

| Rol | Descripción | Necesidad principal |
|-----|-------------|----------------------|
| **Emisor / Issuer (Backoffice)** | Crea y gestiona el proyecto, el escrow y la tokenización. | Gestionar escrows, tokenizar un escrow, desplegar venta de tokens y vault, seguir hitos y liberaciones. |
| **Inversor (Investor)** | Compra tokens con USDC y reclama ROI. | Ver ofertas, comprar tokens, ver saldos y reclamar retornos según su participación. |
| **Lector / Transparencia (Project Updates)** | Consulta el estado del proyecto sin operar. | Ver avance de hitos, estado del escrow y ciclo de vida del proyecto. |

---

## 3. Viajes principales (a nivel de producto)

### 3.1 Emisor: de escrow a token listo para vender

1. **Crear y configurar el escrow** (multi-release, hitos, roles) en Trustless Work.
2. **Tokenizar el escrow:** indicar qué escrow se tokeniza y definir nombre y símbolo del token. El sistema despliega:
   - Un **token** (Token Factory) vinculado de forma inmutable a ese escrow.
   - Un **Token Sale** que recibe USDC y envía los fondos al escrow; solo este contrato puede mintear tokens.
3. **Opcional:** desplegar **vault** para concentrar retornos y permitir que los inversores reclamen ROI.
4. A partir de aquí, los inversores pueden **comprar tokens** (USDC → escrow) y, cuando exista vault y retornos, **reclamar ROI** según su balance.

**Documentación técnica del flujo:** [tokenize-escrow-flow.md](./tokenize-escrow-flow.md)

### 3.2 Inversor: participar y obtener retornos

1. **Descubrir proyectos** (por ejemplo, listado de tokens/ventas asociados a escrows).
2. **Comprar tokens:** enviar USDC al Token Sale; recibe tokens y el USDC va al escrow del proyecto.
3. **Ver participación:** consultar **saldos de tokens** por proyecto (lectura desde storage Soroban, sin depender de invocar el contrato en cada vista).
4. **Reclamar ROI:** cuando el vault tenga retornos, reclamar la parte proporcional a su balance de tokens.

**Documentación técnica de saldos:** [TOKEN_BALANCE_SYSTEM.md](./TOKEN_BALANCE_SYSTEM.md)

### 3.3 Transparencia: seguir el proyecto

1. Ver **estado del escrow** y de los hitos (Trustless Work).
2. Ver **avance y actualizaciones** del proyecto (portal de project updates).
3. Entender el **ciclo de vida:** financiación → ejecución → liberaciones → retornos (vault) → claims.

---

## 4. Capacidades de producto (resumen)

- **Tokenización de escrow:** un escrow → un token (nombre y símbolo configurables), con mint exclusivo vía Token Sale.
- **Venta primaria:** compra de tokens con USDC; fondos dirigidos al escrow del proyecto.
- **Saldos de tokens:** lectura de balances desde storage para mostrar participación por proyecto.
- **Metadata de token:** nombre, símbolo, decimales (y en el contrato, `escrow_id`) para identificación y compliance.
- **Vault y ROI:** contrato que acumula retornos y permite claim proporcional al balance de tokens.
- **Tres interfaces:** Backoffice (emisor), Investor (inversor), Project Updates (transparencia).

---

## 5. Restricciones y reglas de producto

- **Un token por escrow:** cada token está ligado de forma inmutable a un `escrow_id`; no se reasigna.
- **Solo el Token Sale mintea:** la capacidad de mintear está fijada en el contrato de venta; el emisor no puede mintear después del despliegue.
- **USDC como medio de pago** en la compra de tokens (definido en el contrato Token Sale).
- **Red:** actualmente el template está pensado para **Soroban Testnet** (configurable por entorno).

---

## 6. Recursos adicionales

| Recurso | Descripción |
|---------|-------------|
| [Trustless Work – Tokenization Private Credit](https://docs.trustlesswork.com/trustless-work/oss-dapps/tokenization-private-credit) | Documentación oficial del producto. |
| [Vídeo Loom – Tokenization Private Credit](https://www.loom.com/share/08e5ef97f94f42d1b80bebac733a6383) | Demostración en vídeo del flujo. |
| [tokenize-escrow-flow.md](./tokenize-escrow-flow.md) | Arquitectura técnica del flujo de tokenización (componentes, secuencia, contratos). |
| [TOKEN_BALANCE_SYSTEM.md](./TOKEN_BALANCE_SYSTEM.md) | Cómo se leen y muestran los saldos de tokens. |
| [README del repositorio](../README.md) | Estructura del monorepo, contratos y apps. |

---

## 7. Glosario breve

- **Escrow (Trustless Work):** contrato de depósito en custodia con hitos, aprobaciones y liberaciones; base del proyecto financiado.
- **Token Factory:** contrato del token de participación; metadata y mint authority inmutables.
- **Token Sale:** contrato que vende tokens a cambio de USDC y envía el USDC al escrow; único minter.
- **Vault:** contrato que recibe retornos del proyecto y permite a los titulares de tokens reclamar ROI según su balance.
- **Tokenización (de un escrow):** crear el par Token + Token Sale asociado a un escrow para permitir compra de participación y enrutamiento de fondos.

---

*Documento de producto. Para detalles de implementación y rutas de código, ver los documentos técnicos enlazados.*
