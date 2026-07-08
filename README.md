# RecSys_G14_2026

Código y resultados del proyecto de IIC3633 — Grupo 14. Este README documenta la estructura del repositorio y los pasos para replicar los experimentos.

## Estructura del repositorio

```
utils.ipynb                  # pipeline común: descarga/parseo de ReDial y PEARL, construcción de contexto, métricas
analisis_datasets.ipynb      # análisis exploratorio de ambos datasets (requiere utils.ipynb)

CoT/                         # baseline Chain-of-Thought
Reflexion/                   # método Reflexion (Shinn et al. 2023)
Dynamic Cheatsheet/          # método Dynamic Cheatsheet (incluye los cheatsheets .txt generados)
ace/                         # método ACE (Agentic Context Engineering)
  ace_base.ipynb             #   ACE sin tools   — DATASET="redial" por defecto
  ace_tools.ipynb            #   ACE + tool TMDB — DATASET="pearl"  por defecto
  eval_ace_pearl.ipynb       #   recarga y evalúa resultados de ACE base ya generados para PEARL
  redial/, pearl/            #   playbooks, resultados y métricas guardados por dataset

judge/                       # scores del juez LLM (Gemini), un archivo por método × dataset (+ *_failed.json)
llm_judge.ipynb              # corre el juez LLM sobre las predicciones de cada método
llm_judge_analysis.ipynb     # consolida judge/*.json y corre estadística (Friedman/Wilcoxon, Kruskal-Wallis)

Datasets/pearl_test.json     # copia cacheada de referencia; el pipeline en realidad descarga los
                              # datasets automáticamente a la carpeta de trabajo (ver más abajo), no lee de aquí
```

Cada carpeta de método guarda, junto al notebook, sus artefactos por dataset: `*_results_{redial,pearl}.json`, `*_metrics_{redial,pearl}.json`, y `*_buffer_*` / `*_playbook_*` cuando el método acumula memoria (Reflexion, Dynamic Cheatsheet, ACE).

## Requisitos

- Todos los notebooks están pensados para correr en **Google Colab** con GPU (una T4 alcanza para Qwen3-8B en 4-bit).
- Google Drive montado, con la carpeta del proyecto accesible como `Mi unidad/Proyecto RecSys` (ver celda de montaje al inicio de cada notebook — ajustar `PROJECT_DIR`/`PATH` si el nombre difiere).
- Dependencias instaladas inline en la primera celda de cada notebook (`transformers`, `accelerate`, `bitsandbytes`, `sentence-transformers`, `evaluate`, `bert_score`, `google-genai`, `pydantic`). No hay `requirements.txt`.
- `utils.ipynb` debe estar en la misma carpeta que el notebook a correr — algunos notebooks lo convierten a `utils.py` automáticamente al importarlo, otros asumen que ya existe.

## API keys necesarias

- **`GEMINI_API_KEY`** — para `llm_judge.ipynb` (juez LLM). En Colab se lee de `userdata.get("GOOGLE_API_KEY")`; en local, variable de entorno.
- **API key de TMDB** — para `ace/ace_tools.ipynb` (metadata de películas ya mencionadas en la conversación). La key que quedó hardcodeada en el notebook fue **revocada**; hay que generar una propia en themoviedb.org y reemplazar `TMDB_API_KEY` en la celda correspondiente antes de correr.

## Cómo correr los experimentos

Todos los métodos comparten el mismo sampling reproducible (`seed=42`, `n_eval=300`, `n_warmup=100` cuando aplica) y las mismas métricas (`threshold=0.90`, `gt_field="gt_accepted"`), definidas en `utils.ipynb`.

1. **`CoT/cot.ipynb`**, **`Reflexion/reflexion.ipynb`**, **`Dynamic Cheatsheet/dynamic_cheatsheet.ipynb`** — cada uno tiene una variable `DATASET = "pearl"` (o `"redial"`) cerca del inicio; correr una vez por valor para cubrir ambos datasets.
2. **`ace/ace_base.ipynb`** y **`ace/ace_tools.ipynb`** — mismo patrón de `DATASET`, con dos detalles a tener en cuenta al cambiar de dataset:
   - En `ace_tools.ipynb`, los nombres de archivo de salida están hardcodeados con prefijo `pearl_` (`pearl_playbook_tools.json`, `pearl_ace_results_tools.json`, `pearl_ace_playbook_tools.json`, `pearl_metrics_tools.json`) — **no** están parametrizados con `DATASET`, así que hay que renombrarlos a mano (p. ej. a `redial_...`) para no sobrescribir una corrida con otra.
   - `ace_base.ipynb` viene configurado para ReDial. La celda "Correr y evaluar" carga un playbook ya generado (`PATH_FILE`) en vez de llamar a `ace_warmup` directamente; para correrlo desde cero (o para PEARL) hay que invocar `ace_warmup(train_parsed, ...)` explícitamente y ajustar ese path.
   - `eval_ace_pearl.ipynb` no genera nada nuevo: solo recarga (`ace_results_base_pearl.json`, `playbook_final_base_pearl.json`, subidos manualmente a `/content/` en la sesión de Colab) y evalúa resultados de ACE base ya generados para PEARL.
3. **`llm_judge.ipynb`** — completar la lista `RUNS` con los `predictions_path` de los método/dataset a evaluar y correr (requiere `GEMINI_API_KEY`). Es checkpointeable: reintenta automáticamente los índices fallidos en `judge/*_failed.json`.
4. **`llm_judge_analysis.ipynb`** — consolida todos los `judge/*.json` de los 5 métodos × 2 datasets generados en el paso anterior y corre las pruebas estadísticas. No llama a ninguna API.
5. **`analisis_datasets.ipynb`** — independiente del resto de los experimentos; solo requiere `utils.ipynb`.

## Uso de IA

Se usó Claude (Anthropic) como apoyo para partes del código y para escribir este README.
