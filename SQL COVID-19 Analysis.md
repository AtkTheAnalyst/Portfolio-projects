# SQL COVID-19 Data Analysis

This file presents the analysis from `SQL COVID-19 Analysis.sql` in a clean, GitHub-friendly format with SQL syntax highlighting.

---

## Skills used

- Joins
- CTEs
- Temp tables
- Window functions
- Aggregate functions
- Creating views
- Converting data types (`CAST`, `CONVERT`)

---

## Starting dataset

Filter out aggregate rows (where `continent` is null) and inspect the `CovidDeaths` table.

```sql
SELECT *
FROM PortfolioProject..CovidDeaths
WHERE continent IS NOT NULL
ORDER BY 3, 4;
```

### Select the starting columns

```sql
SELECT location, date, total_cases, new_cases, total_deaths, population
FROM PortfolioProject..CovidDeaths
WHERE continent IS NOT NULL
ORDER BY 1, 2;
```

---

## Total Cases vs Total Deaths (Death %)

Goal: estimate likelihood of dying if infected (example filtered to locations containing “states”).

```sql
SELECT location,
       date,
       total_cases,
       total_deaths,
       (total_deaths / total_cases) * 100 AS DeathPercentage
FROM PortfolioProject..CovidDeaths
WHERE location LIKE '%states%'
  AND continent IS NOT NULL
ORDER BY 1, 2;
```

---

## Total Cases vs Population (% infected)

Goal: percent of the population infected over time.

```sql
SELECT location,
       date,
       population,
       total_cases,
       (total_cases / population) * 100 AS PercentPopulationInfected
FROM PortfolioProject..CovidDeaths
ORDER BY 1, 2;
```

---

## Countries with highest infection rate vs population

Goal: peak infection count and peak percent infected, by country.

```sql
SELECT location,
       population,
       MAX(total_cases) AS HighestInfectionCount,
       MAX((total_cases / population)) * 100 AS PercentPopulationInfected
FROM PortfolioProject..CovidDeaths
GROUP BY location, population
ORDER BY PercentPopulationInfected DESC;
```

---

## Countries with highest death count

Goal: max deaths by country (casting to integer).

```sql
SELECT location,
       MAX(CAST(total_deaths AS int)) AS TotalDeathCount
FROM PortfolioProject..CovidDeaths
WHERE continent IS NOT NULL
GROUP BY location
ORDER BY TotalDeathCount DESC;
```

---

## Continents with highest death count

Goal: max deaths by continent.

```sql
SELECT continent,
       MAX(CAST(total_deaths AS int)) AS TotalDeathCount
FROM PortfolioProject..CovidDeaths
WHERE continent IS NOT NULL
GROUP BY continent
ORDER BY TotalDeathCount DESC;
```

---

## Global numbers

Goal: global totals and death percentage (summing daily new cases/deaths).

```sql
SELECT SUM(new_cases) AS total_cases,
       SUM(CAST(new_deaths AS int)) AS total_deaths,
       SUM(CAST(new_deaths AS int)) / SUM(new_cases) * 100 AS DeathPercentage
FROM PortfolioProject..CovidDeaths
WHERE continent IS NOT NULL
ORDER BY 1, 2;
```

---

## Total Population vs Vaccinations (rolling total)

Goal: rolling count of people vaccinated using a window function.

```sql
SELECT dea.continent,
       dea.location,
       dea.date,
       dea.population,
       vac.new_vaccinations,
       SUM(CONVERT(int, vac.new_vaccinations))
         OVER (PARTITION BY dea.location ORDER BY dea.location, dea.date) AS RollingPeopleVaccinated
FROM PortfolioProject..CovidDeaths dea
JOIN PortfolioProject..CovidVaccinations vac
  ON dea.location = vac.location
 AND dea.date = vac.date
WHERE dea.continent IS NOT NULL
ORDER BY 2, 3;
```

---

## CTE: Calculate percent vaccinated

Goal: reuse the rolling vaccination calculation and compute percent vaccinated.

```sql
WITH PopvsVac (Continent, Location, Date, Population, New_Vaccinations, RollingPeopleVaccinated) AS
(
  SELECT dea.continent,
         dea.location,
         dea.date,
         dea.population,
         vac.new_vaccinations,
         SUM(CONVERT(int, vac.new_vaccinations))
           OVER (PARTITION BY dea.location ORDER BY dea.location, dea.date) AS RollingPeopleVaccinated
  FROM PortfolioProject..CovidDeaths dea
  JOIN PortfolioProject..CovidVaccinations vac
    ON dea.location = vac.location
   AND dea.date = vac.date
  WHERE dea.continent IS NOT NULL
)
SELECT *,
       (RollingPeopleVaccinated / Population) * 100
FROM PopvsVac;
```

---

## Temp table: Store percent vaccinated

Goal: store results in a temp table (useful for intermediate steps / repeated querying).

```sql
DROP TABLE IF EXISTS #PercentPopulationVaccinated;

CREATE TABLE #PercentPopulationVaccinated
(
  Continent nvarchar(255),
  Location nvarchar(255),
  Date datetime,
  Population numeric,
  New_vaccinations numeric,
  RollingPeopleVaccinated numeric
);

INSERT INTO #PercentPopulationVaccinated
SELECT dea.continent,
       dea.location,
       dea.date,
       dea.population,
       vac.new_vaccinations,
       SUM(CONVERT(int, vac.new_vaccinations))
         OVER (PARTITION BY dea.location ORDER BY dea.location, dea.date) AS RollingPeopleVaccinated
FROM PortfolioProject..CovidDeaths dea
JOIN PortfolioProject..CovidVaccinations vac
  ON dea.location = vac.location
 AND dea.date = vac.date;

SELECT *,
       (RollingPeopleVaccinated / Population) * 100
FROM #PercentPopulationVaccinated;
```

---

## View: Save for visualization

Goal: create a view for later reporting/visualization work.

```sql
CREATE VIEW PercentPopulationVaccinated AS
SELECT dea.continent,
       dea.location,
       dea.date,
       dea.population,
       vac.new_vaccinations,
       SUM(CONVERT(int, vac.new_vaccinations))
         OVER (PARTITION BY dea.location ORDER BY dea.location, dea.date) AS RollingPeopleVaccinated
FROM PortfolioProject..CovidDeaths dea
JOIN PortfolioProject..CovidVaccinations vac
  ON dea.location = vac.location
 AND dea.date = vac.date
WHERE dea.continent IS NOT NULL;
```


