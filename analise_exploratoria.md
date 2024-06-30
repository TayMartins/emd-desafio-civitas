### Requisitos

1. **Este passo permite uma visualização inicial da tabela, verificando quais colunas estão presentes, os tipos  de dados em cada coluna e exemplos de valores. Isso é fundamental para entender a estrutura dos dados e identificar possíveis inconsistências ou problemas de formatação que precisam ser tratados**

SELECT 
    * 
FROM `rj-cetrio.desafio.readings_2024_06` LIMIT 5;

2. **Saber o número total de registros na tabela é crucial para dimensionar o volume de dados que está sendo analisado. Isso ajuda a planejar a análise e identificar possíveis problemas de desempenho ao trabalhar com grandes conjuntos de dados.**

SELECT DISTINCT 
    COUNT(*) AS total_registros
FROM `rj-cetrio.desafio.readings_2024_06`;

3. **Dados faltantes podem indicar problemas de captura ou registro.**

SELECT 
    *
FROM 
    `rj-cetrio.desafio.readings_2024_06`
  WHERE 
    placa IS NULL 
    OR datahora IS NULL 
    OR camera_latitude IS NULL 
    OR camera_longitude IS NULL 
    OR velocidade IS NULL;


4. **Limitar a análise aos dados dentro das coordenadas do Rio de Janeiro garante que a análise seja relevante para a área de interesse. Isso ajuda a filtrar dados irrelevantes que podem distorcer os resultados da análise.**

SELECT 
    *
FROM 
    `rj-cetrio.desafio.readings_2024_06`
WHERE 
    camera_latitude BETWEEN -23.0829 AND -22.7436
    AND camera_longitude BETWEEN -43.7957 AND -43.1246;


5. **Encontrar a velocidade mínima e máxima no dataset. A minima pode trazer algumas incosistencias, por exemplo, se tiver velocidade negativa. E a velocidade maxima serve para ajustarmos o nosso limite dado o contexto específico da análise e as condições reais de tráfego na região monitorada pelos radares**

SELECT 
    MIN(velocidade) AS velocidade_minima,
    MAX(velocidade) AS velocidade_maxima
FROM 
    `rj-cetrio.desafio.readings_2024_06`;


6. **Contar as ocorrências de Velocidade 0 km/h: ajuda a identificar falhas na medição ou situações onde o veículo realmente estava parado. Ja ocorrências de Velocidade 255 km/h pode indicar problemas nos dispositivos de captura ou a necessidade de revisar os dados. Seria interessante saber qual a velocidade maxima da via para saber quantos veiculos passam dessa velocidade. Qual a porcentagem está fora e dentro da margem aceita**

SELECT 
    velocidade,
    COUNT(*) AS ocorrencias
FROM 
   `rj-cetrio.desafio.readings_2024_06`
WHERE 
    velocidade < 0 OR velocidade IN (0, 255)
GROUP BY 
    velocidade;


7. **Verificar as localizações com o maior número de detecções ajuda a identificar áreas com tráfego intenso e possíveis outliers geográficos. Isso é útil para planejar intervenções e melhorar a infraestrutura de trânsito. - Mapa de Calor das Detecções**

SELECT 
    camera_latitude, 
    camera_longitude, 
    COUNT(*) AS num_detecoes
FROM 
   `rj-cetrio.desafio.readings_2024_06`
GROUP BY 
    camera_latitude, 
    camera_longitude
ORDER BY 
    num_detecoes DESC;


8. **O objetivo aqui é identificar padrões de tráfego para que seja possivel melhorar o planejamento e gerenciamento de Tráfego, a segurança no trânsito, a previsão de tráfego e identificar comportamentos atípicos**

SELECT 
    EXTRACT(HOUR FROM datahora) AS hora_dia,
    COUNT(*) AS num_detecoes
FROM 
     `rj-cetrio.desafio.readings_2024_06`
GROUP BY 
    hora_dia
ORDER BY 
    hora_dia;


9. **Calcular estatísticas descritivas, como média, mediana, desvio padrão, mínimo e máximo, fornece uma visão geral das velocidades registradas. Isso ajuda a entender a distribuição dos dados e a identificar anomalias.**

WITH percentiles AS (
    SELECT 
        velocidade,
        PERCENTILE_CONT(velocidade, 0.5) OVER() AS mediana_velocidade
    FROM 
        `rj-cetrio.desafio.readings_2024_06`
)
SELECT 
    AVG(velocidade) AS media_velocidade,
    APPROX_QUANTILES(velocidade, 100)[OFFSET(50)] AS mediana_velocidade,
    STDDEV(velocidade) AS desvio_padrao_velocidade,
FROM 
    `rj-cetrio.desafio.readings_2024_06`;


10. **Analisar a distribuição das detecções por dia da semana ajuda a identificar variações no tráfego ao longo da semana. Isso pode revelar padrões importantes, como aumento de tráfego em dias úteis ou fins de semana, auxiliando no planejamento de intervenções específicas para esses períodos.**

SELECT 
    EXTRACT(DAYOFWEEK FROM datahora) AS dia_semana,
    COUNT(*) AS num_detecoes
FROM 
    `rj-cetrio.desafio.readings_2024_06`
GROUP BY 
    dia_semana
ORDER BY 
    dia_semana;

## A segmentação de dados e a análise de segmentos específicos permitem uma compreensão mais profunda dos padrões de tráfego e dos comportamentos diferenciados. Isso é essencial para tomar decisões informadas e implementar medidas eficazes para melhorar o gerenciamento do tráfego e a segurança viária. Alguns metodos para segmentar:# 

1) **Segmentação por Tipo de Veículo**
**Diferentes tipos de veículos podem ter padrões de velocidade e comportamento distintos. Isso ajuda a identificar quais tipos de veículos estão mais frequentemente associados a velocidades excessivas ou infrações de trânsito.**

SELECT 
    tipoveiculo,
    COUNT(*) AS num_detecoes,
    AVG(velocidade) AS media_velocidade,
    APPROX_QUANTILES(velocidade, 100)[OFFSET(50)] AS mediana_velocidade,
    STDDEV(velocidade) AS desvio_padrao_velocidade
FROM 
    `rj-cetrio.desafio.readings_2024_06`
GROUP BY 
    tipoveiculo
ORDER BY 
    num_detecoes DESC;


2) **Segmentação por Empresa de Radar**
**Agrupa os dados com base na empresa responsável pela instalação e manutenção dos radares. Diferentes empresas podem ter padrões de detecção variados. Isso ajuda a identificar possíveis discrepâncias na calibração e eficácia dos radares de diferentes fornecedores.**

SELECT 
    empresa,
    COUNT(*) AS num_detecoes,
    AVG(velocidade) AS media_velocidade,
    APPROX_QUANTILES(velocidade, 100)[OFFSET(50)] AS mediana_velocidade,
    STDDEV(velocidade) AS desvio_padrao_velocidade
FROM 
    `rj-cetrio.desafio.readings_2024_06`
GROUP BY 
    empresa
ORDER BY 
    num_detecoes DESC;



### Verificar Placas Clonadas

**Para identificar veículos detectados em múltiplos locais ao mesmo tempo, forte indicador de clonagem de placas e também aqueles com padrões temporais suspeitos (ocorrências repetitivas no mesmo minuto);**


WITH deteccoes AS (
    SELECT 
        placa, 
        datahora,
        camera_latitude,
        camera_longitude,
        COUNT(*) OVER (PARTITION BY placa, datahora) AS num_locais_simultaneos,
        EXTRACT(MINUTE FROM datahora) AS minuto,
        COUNT(*) OVER (PARTITION BY placa, EXTRACT(MINUTE FROM datahora)) AS num_ocorrencias_minuto
    FROM 
        `rj-cetrio.desafio.readings_2024_06`
)
SELECT 
    placa, 
    datahora,
    camera_latitude,
    camera_longitude,
    num_locais_simultaneos,
    num_ocorrencias_minuto
FROM 
    deteccoes
WHERE 
    num_locais_simultaneos > 1 -- Critério 1: Locais Simultâneos
    OR num_ocorrencias_minuto > 10 -- Critério 2: Ocorrências no Mesmo Minuto
ORDER BY 
    placa, datahora;


## Razão para a Eficácia desta Técnica
**Ao combinar múltiplos critérios de detecção (múltiplos locais simultâneos e padrões temporais suspeitos), aumentamos a precisão da detecção de fraudes. Cada critério captura diferentes aspectos do comportamento anômalo, proporcionando uma abordagem mais abrangente.**

**Esta técnica é robusta porque não depende de um único tipo de anomalia. Veículos clonados podem apresentar diferentes padrões suspeitos, e esta abordagem captura várias possíveis fraudes.**

**Ao usar múltiplos critérios, a probabilidade de falsos positivos é reduzida. Um veículo legítimo pode ser detectado em dois locais próximos ao mesmo tempo por acaso, mas é menos provável que ele tenha um padrão repetitivo no mesmo minuto.**