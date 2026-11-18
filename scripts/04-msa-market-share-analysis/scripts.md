# 4. SCRIPTS MSA (MARKET SHARE ANALYSIS)

**Nota:** *Los siguientes scripts están diseñados para el análisis de datos orientado a la estimación de market share, la generación de visualizaciones analíticas y la identificación de insights estratégicos a partir de datos históricos obtenidos de TradeMap. Se sugiere implementar este flujo de trabajo en entornos de ejecución Python tales como:*

- [Google Colab](https://colab.research.google.com/)
- [Python-Fiddle](https://python-fiddle.com/)
- [Kaggle Notebooks](https://www.kaggle.com/code)
- [Deepnote](https://deepnote.com/)

## 4.1. Importar e instalar librerías

```
# Instalar librerías si es necesario
!pip install openpyxl

import pandas as pd
import matplotlib.pyplot as plt
import seaborn as sns
import numpy as np
from google.colab import files
import warnings
warnings.filterwarnings('ignore')

# Configurar estilo de gráficos
plt.style.use('seaborn-v0_8')
sns.set_palette("husl")
```

4.2. Cargar archivos fuente (Trademap)
Nota: Ejecutar después de 4.1. Funciona con el Excel de TradeMap en español ("Exportadores", "Valor importado en AAAA") o en inglés ("Exporters", "Value imported in YYYY"), aunque el archivo traiga filas de título antes de los encabezados.
```
import re

# Subir archivo Excel directamente desde TradeMap
uploaded = files.upload()
if not uploaded:
    raise ValueError("No se subió ningún archivo. Ejecuta la celda de nuevo y selecciona el Excel de TradeMap.")
filename = list(uploaded.keys())[0]

# Detectar automáticamente la fila de encabezados (puede haber filas de título arriba)
EXPORTER_NAMES = {'exportadores', 'exporters'}
raw = pd.read_excel(filename, sheet_name=0, header=None)
header_row = None
for i in range(min(len(raw), 30)):
    valores = raw.iloc[i].astype(str).str.strip().str.lower()
    if valores.isin(EXPORTER_NAMES).any():
        header_row = i
        break
if header_row is None:
    raise ValueError("No se encontró la columna 'Exportadores' (o 'Exporters'). Verifica que el archivo sea el Excel descargado de TradeMap.")

df_raw = pd.read_excel(filename, sheet_name=0, header=header_row)
df_raw.columns = [str(c).strip() for c in df_raw.columns]

# Verificar estructura original
print("Columnas disponibles:", df_raw.columns.tolist())
print("\nPrimeras 5 filas:")
print(df_raw.head())

# Identificar columna de países y columnas de años
country_col = next(c for c in df_raw.columns if c.lower() in EXPORTER_NAMES)
year_pattern = re.compile(r'(?:valor importado en|value imported in)\D*(\d{4})', re.IGNORECASE)
year_columns = [c for c in df_raw.columns if year_pattern.search(c)]
if not year_columns:
    raise ValueError("No se encontraron columnas 'Valor importado en AAAA'. Revisa los nombres de columnas impresos arriba.")
print(f"\nColumnas de años encontradas: {year_columns}")

# Transformar de formato ancho a formato largo (melt)
df_melted = df_raw.melt(
    id_vars=[country_col],
    value_vars=year_columns,
    var_name='year_column',
    value_name='trade_value_usd'
)

# Extraer año de la columna y renombrar país
df_melted['year'] = df_melted['year_column'].apply(lambda c: int(year_pattern.search(c).group(1)))
df_melted = df_melted.rename(columns={country_col: 'partner_country'})
df_melted['partner_country'] = df_melted['partner_country'].astype(str).str.strip()

# Limpiar datos
df_melted['trade_value_usd'] = pd.to_numeric(df_melted['trade_value_usd'], errors='coerce')
df = df_melted[['year', 'partner_country', 'trade_value_usd']].copy()

# Eliminar filas con datos faltantes y fila "Mundo"/"World"
df = df.dropna(subset=['trade_value_usd'])
df = df[~df['partner_country'].str.lower().isin(['mundo', 'world', 'nan'])].reset_index(drop=True)

if df.empty:
    raise ValueError("No quedaron datos después de la limpieza. Revisa el archivo cargado.")

print(f"\nDatos procesados: {len(df)} filas")
print(f"Países únicos: {df['partner_country'].nunique()}")
print(f"Años disponibles: {sorted(df['year'].unique())}")
print("\nPrimeros datos procesados:")
print(df.head(10))
```


## 4.3. Calcular Market Share, Ranking y aplicar filtros

```
# Calcular market share por año y país
market_evolution = df.groupby(['year', 'partner_country'])['trade_value_usd'].sum().reset_index()

# Calcular market share total por año
yearly_totals = market_evolution.groupby('year')['trade_value_usd'].sum()
market_evolution['market_share'] = market_evolution.apply(
    lambda x: (x['trade_value_usd'] / yearly_totals[x['year']]) * 100, axis=1
)

# Identificar top países
top_countries = df.groupby('partner_country')['trade_value_usd'].sum().nlargest(8).index

# Filtrar solo top países
market_evolution_top = market_evolution[market_evolution['partner_country'].isin(top_countries)]
```

## 4.4.  Generar Tablas Pivot para análisis

```
# TABLA 1: Market Share por año y país
print("📊 TABLA 1: MARKET SHARE POR AÑO (%)")
print("="*60)
pivot_share = market_evolution_top.pivot(index='year', columns='partner_country', values='market_share')
pivot_share_rounded = pivot_share.round(1)
print(pivot_share_rounded.to_string())

# TABLA 2: Valores de comercio (Millones USD)
print("\n\n💰 TABLA 2: VALORES DE COMERCIO (Millones USD)")
print("="*60)
pivot_value = market_evolution_top.pivot(index='year', columns='partner_country', values='trade_value_usd')
pivot_value_millions = (pivot_value / 1000).round(1)  # Convertir a millones
print(pivot_value_millions.to_string())

# TABLA 3: Crecimiento anual por país
print("\n\n📈 TABLA 3: CRECIMIENTO ANUAL COMPUESTO (CAGR %)")
print("="*60)
growth_data = market_evolution_top.groupby('partner_country').apply(
    lambda x: ((x['trade_value_usd'].iloc[-1] / x['trade_value_usd'].iloc[0]) ** (1/(len(x)-1)) - 1) * 100
).reset_index(name='cagr')
growth_summary = growth_data.set_index('partner_country')['cagr'].round(1).sort_values(ascending=False)
print(growth_summary.to_string())

# TABLA 4: Ranking por año
print("\n\n🏆 TABLA 4: RANKING DE PAÍSES POR AÑO (Top 5)")
print("="*60)
for year in sorted(df['year'].unique()):
    year_data = market_evolution_top[market_evolution_top['year'] == year].nlargest(5, 'market_share')
    print(f"\n{year}:")
    for i, (_, row) in enumerate(year_data.iterrows(), 1):
        print(f"  {i}. {row['partner_country']}: {row['market_share']:.1f}% ({row['trade_value_usd']/1000:.0f}M USD)")
```

4.5. Generar Gráficos analíticos
Nota: Ejecutar después de 4.3 y 4.4. Esta celda calcula su propio CAGR, por lo que también funciona si se usó el ajuste 4.8.3 o 4.8.4. Si solo hay un año de datos, el panel de CAGR se reemplaza por un aviso. Opcionalmente, cambiar la lista `paises_a_destacar` para resaltar valores de otros países en el primer gráfico.
```
# Verificar que existan los datos de las celdas anteriores
for _nombre in ['df', 'market_evolution_top', 'pivot_share', 'pivot_value']:
    if _nombre not in globals():
        raise NameError(f"Falta '{_nombre}'. Ejecuta primero las secciones 4.1 a 4.4.")

years = list(pivot_share.index)

# CAGR calculado con el primer y último año con datos de cada país
def calcular_cagr(serie):
    s = serie.dropna()
    s = s[s > 0]
    if len(s) < 2:
        return np.nan
    n_anios = s.index[-1] - s.index[0]
    if n_anios <= 0:
        return np.nan
    return ((s.iloc[-1] / s.iloc[0]) ** (1 / n_anios) - 1) * 100

cagr = pivot_value.apply(calcular_cagr).dropna().sort_values(ascending=False)

# Países a resaltar con valores en el gráfico de líneas (si no están, se usan los 2 líderes)
paises_a_destacar = [p for p in ['Brasil', 'Colombia'] if p in pivot_share.columns]
if not paises_a_destacar:
    paises_a_destacar = list(pivot_share.iloc[-1].nlargest(2).index)

fig, axes = plt.subplots(2, 2, figsize=(16, 14))

# 1) Evolución del Market Share
pivot_share.plot(kind='line', marker='o', ax=axes[0, 0], linewidth=2, markersize=6)
axes[0, 0].set_title('Market Share Evolution (%)', fontsize=14, fontweight='bold')
axes[0, 0].set_ylabel('Market Share (%)')
axes[0, 0].set_xlabel('')
axes[0, 0].set_xticks(years)
axes[0, 0].grid(True, alpha=0.3)
axes[0, 0].legend(bbox_to_anchor=(1.05, 1), loc='upper left')

for country in paises_a_destacar:
    for year in pivot_share.index:
        value = pivot_share.loc[year, country]
        if not pd.isna(value):
            axes[0, 0].annotate(f'{value:.1f}%', (year, value),
                                textcoords="offset points", xytext=(0, 10),
                                ha='center', fontsize=8)

# 2) Evolución del valor comercializado (Millones USD)
(pivot_value / 1000).plot(kind='line', marker='s', ax=axes[0, 1], linewidth=2, markersize=6)
axes[0, 1].set_title('Trade Value Evolution (Million USD)', fontsize=14, fontweight='bold')
axes[0, 1].set_ylabel('Million USD')
axes[0, 1].set_xlabel('')
axes[0, 1].set_xticks(years)
axes[0, 1].grid(True, alpha=0.3)
axes[0, 1].legend(bbox_to_anchor=(1.05, 1), loc='upper left')

# 3) Market Share del último año
latest_year = df['year'].max()
latest_data = (market_evolution_top[market_evolution_top['year'] == latest_year]
               .nlargest(8, 'market_share')
               .set_index('partner_country')['market_share'])
latest_data.plot(kind='bar', ax=axes[1, 0], color='skyblue')
axes[1, 0].set_title(f'Market Share {latest_year} (%)', fontsize=14, fontweight='bold')
axes[1, 0].set_ylabel('Market Share (%)')
axes[1, 0].set_xlabel('')
axes[1, 0].tick_params(axis='x', rotation=45)
for contenedor in axes[1, 0].containers:
    axes[1, 0].bar_label(contenedor, fmt='%.1f%%', padding=3, fontsize=9)
axes[1, 0].margins(y=0.12)

# 4) CAGR
if len(cagr) > 0:
    cagr.plot(kind='bar', ax=axes[1, 1], color='lightcoral')
    axes[1, 1].set_title('CAGR % (Compound Annual Growth Rate)', fontsize=14, fontweight='bold')
    axes[1, 1].set_ylabel('CAGR (%)')
    axes[1, 1].set_xlabel('')
    axes[1, 1].tick_params(axis='x', rotation=45)
    axes[1, 1].axhline(y=0, color='black', linestyle='-', alpha=0.3)
    for contenedor in axes[1, 1].containers:
        axes[1, 1].bar_label(contenedor, fmt='%.1f%%', padding=3, fontsize=9)
    axes[1, 1].margins(y=0.15)
else:
    axes[1, 1].set_title('CAGR % (Compound Annual Growth Rate)', fontsize=14, fontweight='bold')
    axes[1, 1].text(0.5, 0.5, 'CAGR no disponible:\nse requieren al menos dos años de datos',
                    ha='center', va='center', fontsize=12, transform=axes[1, 1].transAxes)
    axes[1, 1].set_axis_off()

plt.tight_layout()
plt.show()
```
## 4.6. Generar Insights estratégicos o "Descubrimientos relevantes" 

**Nota:** *Reemplazar "producto analizado" (Line 2) y "Producto" (Line 35) por el número de partida y nombre del producto analizado antes de proceder con la ejecución del script.*

```
# Generar insights automáticos
def generate_insights(df, market_evolution_top, product_name="producto analizado"):
    latest_year = df['year'].max()
    first_year = df['year'].min()

    print(f"📊 INSIGHTS ESTRATÉGICOS - {product_name.upper()}")
    print("="*60)

    # Top 3 países
    top3 = market_evolution_top[market_evolution_top['year'] == latest_year].nlargest(3, 'market_share')
    print(f"\n🏆 TOP 3 PAÍSES ({latest_year}):")
    for i, (_, row) in enumerate(top3.iterrows(), 1):
        print(f"  {i}. {row['partner_country']}: {row['market_share']:.1f}% market share")

    # Mayor crecimiento
    growth_analysis = market_evolution_top.groupby('partner_country').apply(
        lambda x: x['trade_value_usd'].iloc[-1] / x['trade_value_usd'].iloc[0] - 1
    ).sort_values(ascending=False)

    print(f"\n📈 MAYOR CRECIMIENTO ({first_year}-{latest_year}):")
    print(f"  {growth_analysis.index[0]}: +{growth_analysis.iloc[0]:.1%}")

    # Oportunidades
    declining = growth_analysis[growth_analysis < 0]
    if len(declining) > 0:
        print(f"\n⚠️  MERCADOS EN DECLIVE:")
        for country, decline in declining.head(3).items():
            print(f"  {country}: {decline:.1%}")

    # Tamaño total del mercado
    total_market = df[df['year'] == latest_year]['trade_value_usd'].sum()
    print(f"\n💰 TAMAÑO TOTAL DEL MERCADO ({latest_year}): ${total_market/1e9:.1f}B USD")

# Define the product_name variable
product_name = "Producto"

# Aplicar análisis con nombre de producto dinámico
generate_insights(df, market_evolution_top, product_name)
```

## 4.7. Generación de Promts para análisis posteriores

**Nota:** *Reemplazar "market_name" (Line 7) por el mercado objetivo (país-destino) antes de proceder con la ejecución del script.*

```
# GENERAR PROMPT AUTOMÁTICO PARA ANÁLISIS
def generate_analysis_prompt(df, market_evolution_top, product_name):
    latest_year = df['year'].max()
    first_year = df['year'].min()

    # Detectar mercado objetivo (desde columnas o contexto)
    market_name = "Estados Unidos"  # Por defecto, se podría dinamizar

    # Generar estadísticas clave
    top3 = market_evolution_top[market_evolution_top['year'] == latest_year].nlargest(3, 'market_share')
    total_market = df[df['year'] == latest_year]['trade_value_usd'].sum()

    prompt = f"""
    Actúa como consultor en inteligencia de mercados y analiza estos datos de market share de {product_name.lower()} en {market_name} ({first_year}-{latest_year}). Proporciona insights estratégicos específicos:

    CONTEXTO DEL MERCADO:
    - Producto: {product_name}
    - Mercado objetivo: {market_name}
    - Período: {first_year}-{latest_year}
    - Tamaño total del mercado ({latest_year}): ${total_market/1000:.0f} millones USD

    TOP 3 LÍDERES ACTUALES ({latest_year}):
    """

    for i, (_, row) in enumerate(top3.iterrows(), 1):
        prompt += f"\n    {i}. {row['partner_country']}: {row['market_share']:.1f}% market share"

    prompt += f"""

    DATOS PARA ANÁLISIS:
    [Pegar aquí las 4 tablas generadas por el script anterior]

    PREGUNTAS ESPECÍFICAS A RESPONDER:

    1. ANÁLISIS COMPETITIVO:
       - ¿Cuáles son las tendencias de los líderes del mercado?
       - ¿Qué países están ganando/perdiendo participación?
       - ¿Hay oportunidades por concentración/fragmentación del mercado?

    2. OPORTUNIDADES DE ENTRADA:
       - ¿Qué nichos o segmentos presentan menos competencia?
       - ¿Existen países que han reducido su participación creando vacíos?
       - ¿Cuál sería el mejor timing para entrar al mercado?

    3. BENCHMARKING REGIONAL:
       - ¿Cómo se comportan países con perfil similar al de tu empresa?
       - ¿Qué países de América Latina tienen mejor performance?
       - ¿Cuáles son las mejores prácticas observables?

    4. ESTRATEGIA DE POSICIONAMIENTO:
       - ¿Competir directamente con líderes o buscar nichos?
       - ¿Qué ventajas competitivas serían más relevantes?
       - ¿Cuál sería un market share objetivo realista en 2-3 años?

    5. RIESGOS Y AMENAZAS:
       - ¿Qué competidores emergentes representan amenazas?
       - ¿Hay signos de saturación o madurez del mercado?
       - ¿Qué factores externos podrían afectar la industria?

    FORMATO DE RESPUESTA:
    - Insights concisos y accionables
    - Recomendaciones estratégicas específicas
    - Identificación de 2-3 oportunidades prioritarias
    - Métricas clave a monitorear
    """

    return prompt

# Generar y mostrar el prompt
analysis_prompt = generate_analysis_prompt(df, market_evolution_top, product_name)
print("="*80)
print("📋 PROMPT PARA ANÁLISIS ESTRATÉGICO:")
print("="*80)
print(analysis_prompt)
print("\n" + "="*80)
print("💡 INSTRUCCIONES:")
print("1. Copia este prompt")
print("2. Pega las 4 tablas de datos generadas arriba")
print("3. Úsalo con cualquier chatbot para obtener insights estratégicos")
print("="*80)
```

## 4.8. Helpers (opcionales) para depuración y personalización
### 4.8.1. Promt para análisis de oportunidad: Origen (Proveedor)-Destino

```
Con base en lo anterior, redacta un texto argumentativo, en tono formal y académico en tercera persona, máximo una cuartilla (250 palabras) donde evidencies los resultados de este estudio de mercados. Enfatiza en el nivel de oportunidad que tendría [COLOMBIA] para ingresar a este mercado desde el punto de vista comercial basándose únicamente en los datos suministrados.
```

### 4.8.2. Promt para personalizar colores en los gráficos

```
Con base en la siguiente paleta de colores:

[CSS.CODE]

Ajusta el siguiente trozo de código en consecuencia, da por hecho que las librerías ya se encuentran instaladas:

[PY.CODE.4.5]
```

### 4.8.3. Ajuste para datos faltantes o sin datos históricos (No calcula CAGR)

```
# TABLA 1: Market Share por año y país
print("📊 TABLA 1: MARKET SHARE POR AÑO (%)")
print("="*60)
pivot_share = market_evolution_top.pivot(index='year', columns='partner_country', values='market_share')
pivot_share_rounded = pivot_share.round(1)
print(pivot_share_rounded.to_string())

# TABLA 2: Valores de comercio (Millones USD)
print("\n\n💰 TABLA 2: VALORES DE COMERCIO (Millones USD)")
print("="*60)
pivot_value = market_evolution_top.pivot(index='year', columns='partner_country', values='trade_value_usd')
pivot_value_millions = (pivot_value / 1000).round(1)  # Convertir a millones
print(pivot_value_millions.to_string())

# TABLA 3: Crecimiento anual por país
# El cálculo del CAGR requiere al menos dos años de datos. Como solo se tiene un año (2024),
# esta tabla no se puede generar con los datos actuales.
# print("\n\n📈 TABLA 3: CRECIMIENTO ANUAL COMPUESTO (CAGR %)")
# print("="*60)
# growth_data = market_evolution_top.groupby('partner_country').apply(
#     lambda x: ((x['trade_value_usd']. iloc[-1] / x['trade_value_usd'].iloc[0]) ** (1/(len(x)-1)) - 1) * 100
# ).reset_index(name='cagr')
# growth_summary = growth_data.set_index('partner_country')['cagr'].round(1).sort_values(ascending=False)
# print(growth_summary.to_string())
print("\n\n📈 TABLA 3: CRECIMIENTO ANUAL COMPUESTO (CAGR %)")
print("="*60)
print("No se puede calcular el CAGR. Se requiere al menos dos años de datos para este cálculo.")

# TABLA 4: Ranking por año
print("\n\n🏆 TABLA 4: RANKING DE PAÍSES POR AÑO (Top 5)")
print("="*60)
for year in sorted(df['year'].unique()):
    year_data = market_evolution_top[market_evolution_top['year'] == year].nlargest(5, 'market_share')
    print(f"\n{year}:")
    for i, (_, row) in enumerate(year_data.iterrows(), 1):
        print(f"  {i}. {row['partner_country']}: {row['market_share']:.1f}% ({row['trade_value_usd']/1000:.0f}M USD)")
```

### 4.8.4. Ajuste para datos faltantes (Calcula CAGR omitiendo datos vacíos o nulos)

```
# TABLA 1: Market Share por año y país
print("📊 TABLA 1: MARKET SHARE POR AÑO (%)")
print("="*60)
pivot_share = market_evolution_top.pivot(index='year', columns='partner_country', values='market_share')
pivot_share_rounded = pivot_share.round(1)
print(pivot_share_rounded.to_string())

# TABLA 2: Valores de comercio (Millones USD)
print("\n\n💰 TABLA 2: VALORES DE COMERCIO (Millones USD)")
print("="*60)
pivot_value = market_evolution_top.pivot(index='year', columns='partner_country', values='trade_value_usd')
pivot_value_millions = (pivot_value / 1000).round(1)  # Convertir a millones
print(pivot_value_millions.to_string())

# TABLA 3: Crecimiento anual por país
print("\n\n📈 TABLA 3: CRECIMIENTO ANUAL COMPUESTO (CAGR %)")
print("="*60)
growth_data = market_evolution_top.groupby('partner_country').apply(
    lambda x: ((x['trade_value_usd'].iloc[-1] / x['trade_value_usd'].iloc[0]) ** (1/(len(x)-1)) - 1) * 100 if len(x) > 1 else np.nan
).reset_index(name='cagr')
growth_summary = growth_data.set_index('partner_country')['cagr'].round(1).sort_values(ascending=False)
print(growth_summary.to_string())

# TABLA 4: Ranking por año
print("\n\n🏆 TABLA 4: RANKING DE PAÍSES POR AÑO (Top 5)")
print("="*60)
for year in sorted(df['year'].unique()):
    year_data = market_evolution_top[market_evolution_top['year'] == year].nlargest(5, 'market_share')
    print(f"\n{year}:")
    for i, (_, row) in enumerate(year_data.iterrows(), 1):
        print(f"  {i}. {row['partner_country']}: {row['market_share']:.1f}% ({row['trade_value_usd']/1000:.0f}M USD)")
```
