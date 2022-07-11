# Análise Exploratória de Dados da COVID 19 utilizando linguagem SQL

## Total de registros
SELECT COUNT(*) FROM covid.covid_mortes;

SELECT COUNT(*) FROM covid.covid_vacinacao;

## Ordenando por nome de coluna ou número da coluna
SELECT * FROM covid.covid_mortes ORDER BY location, date;

SELECT * FROM covid.covid_mortes ORDER BY location DESC, date;

SELECT * FROM covid.covid_mortes ORDER BY 3,4;

SELECT * FROM covid.covid_vacinacao ORDER BY 3,4;

## Vamos alterar a data para o formato adequado
SET SQL_SAFE_UPDATES = 0;

UPDATE covid.covid_mortes 
SET date = str_to_date(date,'%d/%m/%y');

UPDATE covid.covid_vacinacao 
SET date = str_to_date(date,'%d/%m/%y');

SET SQL_SAFE_UPDATES = 1;

## Retornando algumas colunas relevantes para nosso estudo
SELECT date,
       location,
       total_cases,
       new_cases,
       total_deaths,
       population 

FROM covid.covid_mortes 

ORDER BY 2,1;

## Qual a média de mortos por país?
SELECT location,
       AVG(total_deaths) AS MediaMortos

FROM covid.covid_mortes 

GROUP BY location

ORDER BY MediaMortos DESC;


## Qual a proporção de mortes em relação ao total de casos no Brasil?

SELECT date,
       location, 
       total_cases,
       total_deaths,
       (total_deaths / total_cases) * 100 AS PercentualMortes

FROM covid.covid_mortes  

WHERE location = "Brazil" 

ORDER BY 2,1;

## Qual a proporção média entre o total de casos e a população de cada localidade?
SELECT location,
       AVG((total_cases / population) * 100) AS PercentualPopulacao

FROM covid.covid_mortes  

GROUP BY location

ORDER BY PercentualPopulacao DESC;

## Considerando o maior valor do total de casos, quais os países com a maior taxa de infecção em relação à população?
SELECT location, 
       MAX(total_cases) AS MaiorContagemInfec,
       MAX((total_cases / population)) * 100 AS PercentualPopulacao

FROM covid.covid_mortes 

WHERE continent IS NOT NULL 

GROUP BY location, population 

ORDER BY PercentualPopulacao DESC;


## Quais os países com o maior número de mortes?
SELECT location, 
       MAX(CAST(total_deaths AS UNSIGNED)) AS MaiorContagemMortes

FROM covid.covid_mortes 

WHERE continent IS NOT NULL 

GROUP BY location

ORDER BY MaiorContagemMortes DESC;

## Quais os continentes com o maior número de mortes?
SELECT continent, 
       MAX(CAST(total_deaths AS UNSIGNED)) as MaiorContagemMortes

FROM covid.covid_mortes 

WHERE continent IS NOT NULL 

GROUP BY continent 

ORDER BY MaiorContagemMortes DESC;

## Qual o percentual de mortes por dia?
SELECT date,
       
       SUM(new_cases) as total_cases, 
       
       SUM(CAST(new_deaths AS UNSIGNED)) as total_deaths, 
       
       (SUM(CAST(new_deaths AS UNSIGNED)) / SUM(new_cases)) * 100 as PercentMortes

FROM covid.covid_mortes 

WHERE continent IS NOT NULL 

GROUP BY date 

ORDER BY 1,2;


## Qual o número de novos vacinados e a média móvel de novos vacinados ao longo do tempo na América do Sul?
SELECT mortos.continent,
       mortos.location,
       mortos.date,
       vacinados.new_vaccinations,
       
       AVG(CAST(vacinados.new_vaccinations AS UNSIGNED)) OVER (PARTITION BY mortos.location ORDER BY mortos.date) as MediaMovelVacinados

FROM covid.covid_mortes mortos 

JOIN covid.covid_vacinacao vacinados

ON mortos.location = vacinados.location 

AND mortos.date = vacinados.date

WHERE mortos.continent = 'South America'

ORDER BY 2,3;

## Qual o número de novos vacinados e o total de novos vacinados ao longo do tempo na América do Sul?
SELECT mortos.continent,
       mortos.date,
       vacinados.new_vaccinations,
       
       SUM(CAST(vacinados.new_vaccinations AS UNSIGNED)) OVER (PARTITION BY mortos.continent ORDER BY mortos.date) as TotalVacinados

FROM covid.covid_mortes mortos 

JOIN covid.covid_vacinacao vacinados

ON mortos.location = vacinados.location 

AND mortos.date = vacinados.date

WHERE mortos.continent = 'South America'
ORDER BY 1,2;


## Qual o percentual da população com pelo menos 1 dose da vacina ao longo do tempo no Brasil?
 
WITH PopvsVac (continent,location, date, population, new_vaccinations, TotalMovelVacinacao) AS
(
SELECT mortos.continent,
       mortos.location,
       mortos.date,
       mortos.population,
       vacinados.new_vaccinations,
       
       SUM(CAST(vacinados.new_vaccinations AS UNSIGNED)) OVER (PARTITION BY mortos.location ORDER BY mortos.date) AS TotalMovelVacinacao

FROM covid.covid_mortes mortos 

JOIN covid.covid_vacinacao vacinados 

ON mortos.location = vacinados.location 

AND mortos.date = vacinados.date

WHERE mortos.location = 'Brazil'
)

SELECT *, (TotalMovelVacinacao / population) * 100 AS Percentual_1_Dose FROM PopvsVac;

## Durante o mês de Maio/2021 o percentual de vacinados com pelo menos uma dose aumentou ou diminuiu no Brasil?
WITH PopvsVac (continent, location, date, population, new_vaccinations, TotalMovelVacinacao) AS
(

SELECT mortos.continent,
       mortos.location,
       mortos.date,
       mortos.population,
       vacinados.new_vaccinations,
       
       SUM(CAST(vacinados.new_vaccinations AS UNSIGNED)) OVER (PARTITION BY mortos.location ORDER BY mortos.date) AS TotalMovelVacinacao

FROM covid.covid_mortes mortos 

JOIN covid.covid_vacinacao vacinados 

ON mortos.location = vacinados.location 

AND mortos.date = vacinados.date

WHERE mortos.location = 'Brazil'
)

SELECT *, (TotalMovelVacinacao / population) * 100 AS Percentual_1_Dose 
FROM PopvsVac

WHERE DATE_FORMAT(date, "%M/%Y") = 'May/2021'
AND location = 'Brazil';

## Criando uma VIEW para armazenar a consulta e entregar o resultado
CREATE OR REPLACE VIEW covid.PercentualPopVac AS 
WITH PopvsVac (continent, location, date, population, new_vaccinations, TotalMovelVacinacao) AS
(

SELECT mortos.continent,
       mortos.location,
       mortos.date,
       mortos.population,
       vacinados.new_vaccinations,
       
       SUM(CAST(vacinados.new_vaccinations AS UNSIGNED)) OVER (PARTITION BY mortos.location ORDER BY mortos.date) AS TotalMovelVacinacao

FROM covid.covid_mortes mortos 

JOIN covid.covid_vacinacao vacinados 

ON mortos.location = vacinados.location 

AND mortos.date = vacinados.date

WHERE mortos.location = 'Brazil'
)

SELECT *, (TotalMovelVacinacao / population) * 100 AS Percentual_1_Dose 
FROM PopvsVac
WHERE location = 'Brazil';

## Total de vacinados com pelo menos 1 dose ao longo do tempo
SELECT * FROM covid.PercentualPopVac;

## Total de vacinados com pelo menos 1 dose em Junho/2021
SELECT * FROM covid.PercentualPopVac WHERE DATE_FORMAT(date, "%M/%Y") = 'June/2021';

## Dias com percentual de vacinados com pelo menos 1 dose maior que 30
SELECT * FROM covid.PercentualPopVac WHERE Percentual_1_Dose > 30;
