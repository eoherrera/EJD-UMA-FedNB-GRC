# EJD-UMA-FedNB-GRC
## Aprendizaje Federado con Gobernanza Institucional GRC para Deteccion de Intrusiones

**Autor:** Edgar O. Herrera Logrono, M.Sc. en Inteligencia Artificial, VIU Espana  
**Programa:** Doctorado en Tecnologias Informaticas, Universidad de Malaga  
**Version:** v1.0 | **Fecha:** Mayo 2026  
**Entorno:** Google Colab (Python 3.10+)

---

## El problema que resuelve este trabajo

Un banco, un hospital y una entidad de gobierno quieren entrenar detectores de intrusiones juntos sin compartir sus datos de red. El aprendizaje federado permite eso: cada institucion entrena localmente y el servidor combina lo aprendido.

Pero hay un problema que nadie habia abordado: no todas las instituciones tienen el mismo nivel de madurez en seguridad. Si el servidor combina los modelos por igual, esta ignorando informacion valiosa que ya existe en los marcos de gobernanza que las instituciones usan para auditarse.

Esta investigacion demuestra que usar esa informacion de gobernanza como guia matematica para combinar los modelos produce mejores resultados que simplemente promediar todo por igual.

---

## La idea central: CRISC como regularizador

El marco CRISC de ISACA define variables que miden la madurez de seguridad de cada institucion. Este programa las convierte en un Indice de Coherencia Contextual (ICC) que regula el peso de cada nodo en el servidor federado:

```
ICC = (CMM/5) * KCI * (1 - KRI) * (1 - CVSS/10)
```

| Variable | Significado |
|---|---|
| CMM | Madurez del proceso de gestion de riesgos (1 a 5) |
| KCI | Proporcion de controles de seguridad implementados (0 a 1) |
| KRI | Frecuencia de activacion de alertas, menor es mejor (0 a 1) |
| CVSS | Puntuacion media de vulnerabilidades, menor es mejor (0 a 10) |

Los tres nodos institucionales del experimento y sus ICC calculados:

| Nodo | CMM | KCI | KRI | CVSS | ICC |
|---|---|---|---|---|---|
| Financiero | 4 | 0.82 | 0.12 | 3.2 | 0.393 |
| Salud | 3 | 0.70 | 0.25 | 5.1 | 0.154 |
| Gobierno | 2 | 0.55 | 0.40 | 6.8 | 0.042 |

El servidor aprende cuanto pesar cada nodo usando el optimizador Nelder-Mead con regularizacion L2 sobre el prior ICC. Sin ICC, el servidor daria 33% a cada nodo sin importar su madurez. Con ICC, el nodo Financiero (el mas maduro) recibe consistentemente el mayor peso.

---

## Flujo del programa

```
[Parametros CRISC] CMM, KCI, KRI, CVSS -> ICC por nodo
        |
        v
[Carga de datos] NSL-KDD (2009) | CIC-IDS2017 (2017) | UNSW-NB15 (2015)
  Limpieza, auditoria, submuestreo estratificado
        |
        v
[Preprocesado hibrido]
  Categoricas -> OrdinalEncoder + CategoricalNB (slot OOD activo)
  Numericas   -> StandardScaler + GaussianNB
  Ajuste SOLO sobre train, sin fuga de datos
        |
        v
[Distribucion Dirichlet(alpha)] -> 3 particiones por nodo
  7 niveles de heterogeneidad: alpha en {0.05, 0.1, 0.2, 0.3, 0.5, 0.7, 1.0}
        |
        v
[Entrenamiento local] NaiveBayesHibrido en cada nodo
        |
        v
[Servidor MoG] combina log-verosimilitudes (mixtura real, no promedio de parametros)
  Nelder-Mead aprende pesos con regularizacion ICC
        |
        v
[Evaluacion] F1-macro | ANLL | McNemar | ICC Alignment
        |
        v
[5 Figuras numeradas] Gradiente | ICC Alignment | Pesos | Cruzado | Densidades MoG
        |
        v
[PROTOCOLO-STRESS] 15 verificaciones automaticas
  Si alguna falla el programa detiene y no genera conclusiones
```

---

## Cuatro propuestas comparadas

| Codigo | Nombre | Descripcion |
|---|---|---|
| C | Centralizado | Un modelo con todos los datos. Techo teorico. |
| B | Baseline FedAvg | Pesos proporcionales al tamano del nodo. |
| E | Entropia | Mayor peso al nodo con menor incertidumbre. |
| **A** | **Mezcla ICC (CRISC)** | **Pesos aprendidos por Nelder-Mead con prior ICC. Propuesta principal.** |

---

## Resultados

| Dataset | Ano | A (ICC+CRISC) | B (Baseline) | Delta | McNemar sig |
|---|---|---|---|---|---|
| NSL-KDD | 2009 | 0.9135 | 0.9076 | +0.0059 | 6/7 alphas |
| CIC-IDS2017 | 2017 | 0.7556 | 0.6771 | +0.0785 | 6/7 alphas |
| UNSW-NB15 | 2015 | 0.2110 | 0.2060 | +0.0050 | 1/7 alphas |
| **Promedio** | | **0.6049** | **0.5777** | **+0.0272** | **70/94 (74%)** |

La figura central del trabajo (Fig. 4) muestra el ICC Alignment cruzado en los tres datasets. El patron se reproduce: el nodo Financiero (ICC=0.393) recibe el mayor peso aprendido y el nodo Gobierno (ICC=0.042) el menor, en datasets construidos con ocho anos de diferencia por equipos en tres continentes distintos.

---

## Como ejecutar

1. Abrir `EJD_UMA_FedNB_GRC_v1.0.ipynb` en Google Colab
2. Subir `kaggle.json` cuando lo solicite la primera celda
3. Ejecutar **Runtime > Run all**
4. Tiempo estimado: 90 minutos

Los datasets se descargan automaticamente. No se requiere instalacion previa.

---

## Verificacion de integridad

El programa incluye 15 verificaciones automaticas (PROTOCOLO-STRESS) que se ejecutan antes de generar conclusiones. Resultado actual: **15/15 aprobadas**.

---

## Estructura (31 celdas en orden obligatorio)

```
Kaggle > Sec.0 (parametros) > Sec.1 (entorno) > Sec.2 (carga+limpieza) >
Sec.3 (auditoria) > Sec.4 (verificacion) > Sec.5 (preprocesado) >
Sec.6 (modelo federado) > Sec.7 (metricas) > Sec.8 (experimento) >
Sec.9 (figuras) > Protocolo-Stress > Resumen > Conclusiones
```

---

*Investigacion doctoral en curso. Codigo disponible para revision academica.*
