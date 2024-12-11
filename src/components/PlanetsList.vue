<template>
  <div class="flex flex-col">
    <h1>Star Wars Planets</h1>
    <div class="filter-container">
      <div class="flex flex-row items-center mb-2">
        <label for="search-input">Search by Planet or Resident Name:</label>
        <input
          class="ml-2 p-1 rounded-md text-black"
          id="search-input"
          type="text"
          v-model="searchTerm"
          placeholder="Enter a planet or resident name"
        />
      </div>
      <div class="flex flex-row items-center">
        <label for="terrain-dropdown">Filter by Terrain:</label>
        <select
          class="ml-2 p-1 rounded-md text-black"
          id="terrain-dropdown"
          v-model="selectedTerrain"
        >
          <option value="">All Terrains</option>
          <option v-for="terrain in uniqueTerrains" :key="terrain" :value="terrain">
            {{ terrain }}
          </option>
        </select>
      </div>
    </div>

    <p v-if="isLoading">Loading planets and residents...</p>
    <p v-if="error" class="error">{{ error }}</p>

    <ul v-if="!isLoading && filteredPlanets.length > 0">
      <li class="planet" v-for="planet in filteredPlanets" :key="planet.name">
        <div class="list-item">
          <h2 class="font-bold">{{ planet.name }}</h2>
          <br />
          <strong>Population:</strong> {{ planet.population }}
          <br />
          Appears in <strong>{{ planet.films.length }}</strong> films
          <br />
          <strong>Terrain:</strong> {{ planet.terrain }}
          <br />
          <strong>Climate:</strong> {{ planet.climate }}
          <br />
          <strong>Notable Residents:</strong>
          <br />
          <ul class="residents-list">
            <li v-for="resident in planet.residents" :key="resident">
              {{ resident }}
            </li>
          </ul>
          <em>Last Edited: {{ planet.edited }}</em>
        </div>
        <div class="pr-2">
          <svg width="100" height="100">
            <circle
              :cx="50"
              :cy="50"
              :r="getRadius(diameterToNumber(planet.diameter))"
              :fill="getClimateColor(planet.climate)"
              stroke="black"
              stroke-width="2"
            />
          </svg>
        </div>
      </li>
    </ul>

    <p v-if="!isLoading && filteredPlanets.length === 0">No planets match the search or filter.</p>
  </div>
</template>

<script lang="ts">
import { defineComponent } from 'vue'

interface Planet {
  name: string
  population: string | number
  terrain: string
  climate: string
  residents: string[]
  films: string[]
  diameter: string | number | null
  edited: string
  [key: string]: unknown
}

export default defineComponent({
  name: 'PlanetsList',
  data() {
    return {
      planets: [] as Planet[],
      searchTerm: '',
      selectedTerrain: '',
      isLoading: false,
      error: null as string | null,
    }
  },
  computed: {
    uniqueTerrains(): string[] {
      // Extract unique terrains from the planets list
      const terrains = this.planets.flatMap((planet) =>
        planet.terrain.split(',').map((t) => t.trim()),
      )
      return Array.from(new Set(terrains)).sort() // Remove duplicates and sort
    },

    filteredPlanets(): Planet[] {
      const terrainFilter = this.selectedTerrain.trim().toLowerCase()
      const searchFilter = this.searchTerm.trim().toLowerCase()
      return this.planets.filter((planet) => {
        // Check if terrain matches
        const matchesTerrain = terrainFilter
          ? planet.terrain.toLowerCase().includes(terrainFilter)
          : true

        // If terrain doesn't match, exclude the planet
        if (!matchesTerrain) return false

        // Check if planet name or residents match the search term
        const matchesSearch = searchFilter
          ? planet.name.toLowerCase().includes(searchFilter) ||
            planet.residents.some((resident: string) =>
              resident.toLowerCase().includes(searchFilter),
            )
          : true

        // Include the planet only if both conditions are true
        return matchesSearch
      })
    },
  },
  methods: {
    // Fetches all planets and their residents
    async fetchAllPlanets() {
      this.isLoading = true
      this.error = null
      const allPlanets: Planet[] = []
      let pendingRequests = 0 // Ensures loading state for all pending requests
      let url: string | null = 'https://swapi.dev/api/planets/'

      try {
        while (url) {
          pendingRequests++
          const response = await fetch(url)
          pendingRequests--
          if (!response.ok) {
            throw new Error(`Failed to fetch: ${response.statusText}`)
          }
          const data = await response.json()
          allPlanets.push(...data.results)
          url = data.next // Fetch next page URL due to the pagination meaning there is 10 results per page
        }

        allPlanets.forEach(async (planet) => {
          planet.population =
            planet.population === 'unknown' ? 'unknown' : parseInt(planet.population as string, 10)
          planet.diameter =
            planet.diameter === 'unknown' ? null : parseInt(planet.diameter as string, 10)
          planet.edited = new Date(planet.edited).toLocaleString('en-GB', { timeZone: 'UTC' })
          // Fetch resident names
          if (planet.residents.length > 0) {
            pendingRequests++
            const residentNames = await Promise.all(
              planet.residents.map(async (residentUrl: string) => {
                try {
                  const res = await fetch(residentUrl)
                  const residentData = await res.json()
                  console.log(residentData.name)

                  return residentData.name // Return the resident's name
                } catch {
                  return 'Unknown Resident' // Fallback if a resident fetch fails
                } finally {
                }
              }),
            )
            pendingRequests--

            planet.residents = [...residentNames] // Replace URLs with names
          } else {
            planet.residents = ['No notable residents']
          }
        })

        // Sort planets by population (highest to lowest)
        this.planets = [
          ...allPlanets.sort((a, b) => {
            const popA = a.population === 'unknown' ? -1 : (a.population as number)
            const popB = b.population === 'unknown' ? -1 : (b.population as number)
            return popB - popA
          }),
        ]
      } catch (err: unknown) {
        if (err instanceof Error) return err.message
        return String(err)
      } finally {
        // Ensure loading stops when all requests finish
        const checkLoadingState = setInterval(() => {
          if (pendingRequests === 0) {
            this.isLoading = false
            clearInterval(checkLoadingState)
          }
        }, 50) // Check every 50ms
      }
    },

    // Returns a a radius of between 10 and 45 depending on planet size
    getRadius(diameter: number | null): number {
      if (!diameter || diameter === 0) return 10
      return Math.min(Math.max(diameter / 300, 10), 45)
    },
    getClimateColor(climate: string): string {
      const climateMap: Record<string, string> = {
        //TODO expand colours and make sure they pass accessability standards
        arid: '#D13C1C',
        temperate: '#229954 ',
        tropical: '#D7EA15',
        frozen: '#2E86C1',
        unknown: '#DADADA',
      }
      return climateMap[climate.split(',')[0].trim()] || '#DADADA'
    },

    // Converts any string value to number|null to ensure that the get radius function handles the diameter value correctly as the diameter comes in string format from the API
    diameterToNumber(diameter: string | number | null): number | null {
      if (typeof diameter === 'number') return diameter
      if (typeof diameter === 'string' && diameter !== 'unknown') {
        const parsed = parseInt(diameter, 10)
        return isNaN(parsed) ? null : parsed
      }
      return null
    },
  },
  mounted() {
    this.fetchAllPlanets()
  },
})
</script>
