<script setup>
import { ref, watch, computed } from 'vue'

const currentHour = new Date().toLocaleString('sv').slice(0, 14) + '00';


const cities = [
  {
    name: "Reutlingen",
    lan: 48.49144,
    lon: 9.20427
  },
  {
    name: "Detroit",
    lan: 42.33143,
    lon: -83.04575
  }
];

const searchText = ref('')
const searchResults = ref([])
const selectedCity = ref(cities[0]);
const weather = ref(null);

const selectCity = (event) => {
  const index = event.target.value;
  selectedCity.value = cities[index];
}


// search

const searchCity = async () => {
  const response = await fetch(
    `https://geocoding-api.open-meteo.com/v1/search?name=${encodeURIComponent(searchText.value)}&count=8&language=de&format=json`
  )

  const data = await response.json()

  if (!data.results || data.results.length === 0) {
    searchResults.value = []
    return
  }

  searchResults.value = data.results

}

const selectSearchCity = (city) => {
  selectedCity.value = {
    name: city.name,
    lan: city.latitude,
    lon: city.longitude
  }

  searchText.value = ""
  searchResults.value = []
}



const getCurrentLocation = () => {
  if (!navigator.geolocation) {
    alert('Dein Browser unterstützt keine Standortbestimmung.')
    return
  }

  navigator.geolocation.getCurrentPosition(
    async (position) => {
      const latitude = position.coords.latitude
      const longitude = position.coords.longitude
      try {
        // Koordinaten in Ortsnamen umwandeln
        const response = await fetch(
          `https://nominatim.openstreetmap.org/reverse?lat=${latitude}&lon=${longitude}&format=jsonv2&zoom=10&addressdetails=1&accept-language=de`
        )

        if (!response.ok) {
          throw new Error('Nominatim Anfrage fehlgeschlagen')
        }

        const data = await response.json()
        const address = data.address || {}

        // Den passendsten Ortsnamen auswählen
        const locationName =
          address.city ||
          address.town ||
          address.village ||
          address.municipality ||
          address.hamlet ||
          address.county ||
          'Unbekannter Standort'

        selectedCity.value = {
          name: locationName,
          lan: latitude,
          lon: longitude
        }

      } catch (error) {
        console.error('Fehler beim Ermitteln des Ortsnamens:', error)

        // Standort trotzdem verwenden, falls Nominatim nicht erreichbar ist
        selectedCity.value = {
          name: 'Mein Standort',
          lan: latitude,
          lon: longitude
        }
      }
    },

    (error) => {
      switch (error.code) {
        case error.PERMISSION_DENIED:
          alert('Der Zugriff auf deinen Standort wurde verweigert.')
          break

        case error.POSITION_UNAVAILABLE:
          alert('Dein Standort konnte nicht ermittelt werden.')
          break

        case error.TIMEOUT:
          alert('Die Standortabfrage hat zu lange gedauert.')
          break

        default:
          alert('Der Standort konnte nicht ermittelt werden.')
      }
    },

    {
      enableHighAccuracy: true,
      timeout: 10000,
      maximumAge: 300000
    }
  )
}

const weatherDescriptions = {
  // Klar / Wolken
  0: "Klarer Himmel",
  1: "Überwiegend klar",
  2: "Teilweise bewölkt",
  3: "Bedeckter Himmel",

  // Nebel
  45: "Nebel",
  48: "Nebel mit Reifablagerung",

  // Nieselregen
  51: "Leichter Nieselregen",
  53: "Mäßiger Nieselregen",
  55: "Starker Nieselregen",

  // Gefrierender Nieselregen
  56: "Leichter gefrierender Nieselregen",
  57: "Starker gefrierender Nieselregen",

  // Regen
  61: "Leichter Regen",
  63: "Mäßiger Regen",
  65: "Starker Regen",

  // Gefrierender Regen
  66: "Leichter gefrierender Regen",
  67: "Starker gefrierender Regen",

  // Schnee
  71: "Leichter Schneefall",
  73: "Mäßiger Schneefall",
  75: "Starker Schneefall",
  77: "Schneegriesel",

  // Regenschauer
  80: "Leichte Regenschauer",
  81: "Mäßige Regenschauer",
  82: "Starke Regenschauer",

  // Schneeschauer
  85: "Leichte Schneeschauer",
  86: "Starke Schneeschauer",

  // Gewitter
  95: "Gewitter",
  96: "Gewitter mit leichtem Hagel",
  97: "Starkes Gewitter",
  99: "Gewitter mit starkem Hagel"
};


watch(selectedCity, async function (value) {
  const response = await fetch(`https://api.open-meteo.com/v1/forecast?latitude=${value.lan}&longitude=${value.lon}&hourly=temperature_2m,precipitation_probability,precipitation,weather_code,is_day,cloud_cover&daily=sunrise,sunset,temperature_2m_max,temperature_2m_min,weather_code,precipitation_sum&forecast_days=9&timezone=auto&current=apparent_temperature`);

  weather.value = await response.json();
  console.log(value.lan, value.lon)

}, { immediate: true });

const getLocalTimeForTimezone = (timeZone) => {
  const now = new Date();

  return new Intl.DateTimeFormat('sv-SE', {
    timeZone,
    year: 'numeric',
    month: '2-digit',
    day: '2-digit',
    hour: '2-digit',
    minute: '2-digit',
    hour12: false
  })
    .format(now)
    .replace(' ', 'T');
};

const meteoconIcons = {
  0: {
    day: 'clear-day',
    night: 'clear-night'
  },

  1: {
    day: 'mostly-clear-day',
    night: 'mostly-clear-night'
  },

  2: {
    day: 'partly-cloudy-day',
    night: 'partly-cloudy-night'
  },

  3: {
    day: 'overcast-day',
    night: 'overcast-night'
  },

  45: {
    day: 'fog-day',
    night: 'fog-night'
  },

  48: {
    day: 'fog-day',
    night: 'fog-night'
  },

  51: {
    day: 'overcast-day-drizzle',
    night: 'overcast-night-drizzle'
  },

  53: {
    day: 'overcast-day-drizzle',
    night: 'overcast-night-drizzle'
  },

  55: {
    day: 'overcast-day-drizzle',
    night: 'overcast-night-drizzle'
  },

  56: {
    day: 'overcast-day-drizzle',
    night: 'overcast-night-drizzle'
  },

  57: {
    day: 'overcast-day-drizzle',
    night: 'overcast-night-drizzle'
  },

  61: {
    day: 'mostly-clear-day-rain',
    night: 'mostly-clear-night-rain'
  },

  63: {
    day: 'overcast-day-rain',
    night: 'overcast-night-rain'
  },

  65: {
    day: 'extreme-day-rain',
    night: 'extreme-night-rain'
  },

  66: {
    day: 'overcast-day-rain',
    night: 'overcast-night-rain'
  },

  67: {
    day: 'extreme-day-rain',
    night: 'extreme-night-rain'
  },

  71: {
    day: 'mostly-clear-day-snow',
    night: 'mostly-clear-night-snow'
  },

  73: {
    day: 'overcast-day-snow',
    night: 'overcast-night-snow'
  },

  75: {
    day: 'extreme-day-snow',
    night: 'extreme-night-snow'
  },

  77: {
    day: 'overcast-day-snow',
    night: 'overcast-night-snow'
  },

  80: {
    day: 'partly-cloudy-day-rain',
    night: 'partly-cloudy-night-rain'
  },

  81: {
    day: 'overcast-day-rain',
    night: 'overcast-night-rain'
  },

  82: {
    day: 'extreme-day-rain',
    night: 'extreme-night-rain'
  },

  85: {
    day: 'partly-cloudy-day-snow',
    night: 'partly-cloudy-night-snow'
  },

  86: {
    day: 'extreme-day-snow',
    night: 'extreme-night-snow'
  },

  95: {
    day: 'thunderstorms-day',
    night: 'thunderstorms-night'
  },

  96: {
    day: 'thunderstorms-day-hail',
    night: 'thunderstorms-night-hail'
  },

  97: {
    day: 'extreme-thunderstorms-day',
    night: 'extreme-thunderstorms-night'
  },

  99: {
    day: 'extreme-thunderstorms-day-hail',
    night: 'extreme-thunderstorms-night-hail'
  }
};

const getWeatherIcon = (wmoCode, isDay, precipitation = 0) => {
  const icon = meteoconIcons[wmoCode];

  if (!icon) {
    return new URL(
      '/node_modules/@meteocons/svg/fill/mostly-clear-day.svg',
      import.meta.url
    ).href;
  }

  let iconName = isDay === 1
    ? icon.day
    : icon.night;


  const isThunderstorm = [95, 96, 97, 99].includes(wmoCode);
  const hasRain = precipitation > 0;

  if (isThunderstorm && hasRain) {
    if (wmoCode === 96 || wmoCode === 99) {
      iconName = isDay === 1
        ? 'thunderstorms-day-hail'
        : 'thunderstorms-night-hail';
    } else {
      iconName = isDay === 1
        ? 'thunderstorms-day-rain'
        : 'thunderstorms-night-rain';
    }
  }

  return new URL(
    `/node_modules/@meteocons/svg/fill/${iconName}.svg`,
    import.meta.url
  ).href;
};


const weatherPriority = {
  // Gewitter
  99: 120,
  97: 115,
  96: 110,
  95: 105,

  // Starker Regen / Schauer
  82: 90,
  67: 85,
  65: 80,

  // Starker Schnee
  86: 80,
  75: 75,

  // Mäßiger Regen
  81: 65,
  66: 65,
  63: 60,

  // Schneeschauer
  85: 55,

  // Leichter Regen
  80: 45,
  61: 40,

  // Schnee
  73: 40,
  71: 35,
  77: 35,

  // Nieselregen
  57: 30,
  56: 28,
  55: 25,
  53: 22,
  51: 20,

  // Nebel
  48: 18,
  45: 15,

  // Bewölkung
  3: 12,
  2: 10,
  1: 8,
  0: 5
};



const weatherGroups = {
  thunderstorm: [95, 96, 97, 99],

  heavyRain: [65, 67, 82],

  heavySnow: [75, 86],

  rain: [61, 63, 66, 80, 81],

  snow: [71, 73, 77, 85],

  drizzle: [51, 53, 55, 56, 57],

  fog: [45, 48],

  clouds: [0, 1, 2, 3]
};


const getHoursForCodes = (counts, codes) => {
  return codes.reduce(
    (total, code) => total + (counts[code] || 0),
    0
  );
};


const getIconForPeriod = (indices, isDay) => {
  if (!indices?.length || !weather.value?.hourly) {
    return null;
  }

  const hourly = weather.value.hourly;

  const weatherPeriods = indices
    .map(index => ({
      code: hourly.weather_code[index],
      precipitation: hourly.precipitation[index] ?? 0
    }))
    .filter(
      weather => weather.code !== undefined &&
        weather.code !== null
    );

  if (!weatherPeriods.length) {
    return null;
  }

  const codes = weatherPeriods.map(weather => weather.code);

  const totalHours = codes.length;


  const counts = {};

  codes.forEach(code => {
    counts[code] = (counts[code] || 0) + 1;
  });

  const thunderstormHours = getHoursForCodes(
    counts,
    weatherGroups.thunderstorm
  );

  if (thunderstormHours > 0) {

    const thunderstormWeather = weatherPeriods
      .filter(weather =>
        weatherGroups.thunderstorm.includes(weather.code)
      )
      .sort((a, b) => {

        if (a.precipitation > 0 && b.precipitation <= 0) {
          return -1;
        }

        if (a.precipitation <= 0 && b.precipitation > 0) {
          return 1;
        }

        return (
          (weatherPriority[b.code] || 0) -
          (weatherPriority[a.code] || 0)
        );
      })[0];

    return getWeatherIcon(
      thunderstormWeather.code,
      isDay,
      thunderstormWeather.precipitation
    );
  }


  const heavyRainHours = getHoursForCodes(
    counts,
    weatherGroups.heavyRain
  );

  if (
    heavyRainHours >= 2 &&
    heavyRainHours / totalHours >= 0.25
  ) {

    const code = weatherGroups.heavyRain
      .filter(code => counts[code])
      .sort((a, b) => {
        return (
          (weatherPriority[b] || 0) -
          (weatherPriority[a] || 0)
        );
      })[0];

    return getWeatherIcon(code, isDay);
  }

  const heavySnowHours = getHoursForCodes(
    counts,
    weatherGroups.heavySnow
  );

  if (
    heavySnowHours >= 2 &&
    heavySnowHours / totalHours >= 0.25
  ) {

    const code = weatherGroups.heavySnow
      .filter(code => counts[code])
      .sort((a, b) => {
        return (
          (weatherPriority[b] || 0) -
          (weatherPriority[a] || 0)
        );
      })[0];

    return getWeatherIcon(code, isDay);
  }


  const fogHours = getHoursForCodes(
    counts,
    weatherGroups.fog
  );

  const fogPercentage = fogHours / totalHours;

  if (fogPercentage >= 0.5) {

    const fogCode = weatherGroups.fog
      .filter(code => counts[code])
      .sort((a, b) => {
        return (
          (weatherPriority[b] || 0) -
          (weatherPriority[a] || 0)
        );
      })[0];

    return getWeatherIcon(
      fogCode,
      isDay
    );
  }


  const rainHours = getHoursForCodes(
    counts,
    weatherGroups.rain
  );

  const drizzleHours = getHoursForCodes(
    counts,
    weatherGroups.drizzle
  );

  if (
    rainHours >= 3 &&
    rainHours / totalHours >= 0.30
  ) {

    const rainCodes = weatherGroups.rain
      .filter(code => counts[code]);

    const code = rainCodes.sort((a, b) => {
      return (
        (weatherPriority[b] || 0) -
        (weatherPriority[a] || 0)
      );
    })[0];

    return getWeatherIcon(
      code,
      isDay
    );
  }


  if (rainHours < 3) {
    weatherGroups.rain.forEach(code => {
      delete counts[code];
    });

    weatherGroups.drizzle.forEach(code => {
      delete counts[code];
    });
  }


  const snowHours = getHoursForCodes(
    counts,
    weatherGroups.snow
  );

  if (
    snowHours >= 3 &&
    snowHours / totalHours >= 0.30
  ) {

    const snowCode = weatherGroups.snow
      .filter(code => counts[code])
      .sort((a, b) => {
        return (
          (weatherPriority[b] || 0) -
          (weatherPriority[a] || 0)
        );
      })[0];

    return getWeatherIcon(
      snowCode,
      isDay
    );
  }



  const scores = {};

  Object.entries(counts).forEach(([codeString, count]) => {

    const code = Number(codeString);

    const priority = weatherPriority[code] || 0;

    const percentage = count / totalHours;


    let score = count * priority;


    if (percentage < 0.15) {
      score *= 0.5;
    }

    if (percentage < 0.10) {
      score *= 0.5;
    }

    scores[code] = score;
  });



  const sortedCodes = Object.entries(scores)
    .sort((a, b) => {

      const scoreA = a[1];
      const scoreB = b[1];

      if (scoreA !== scoreB) {
        return scoreB - scoreA;
      }

      const priorityA =
        weatherPriority[Number(a[0])] || 0;

      const priorityB =
        weatherPriority[Number(b[0])] || 0;

      return priorityB - priorityA;
    });


  if (!sortedCodes.length) {
    return getWeatherIcon(
      codes[0],
      isDay
    );
  }


  const selectedCode = Number(
    sortedCodes[0][0]
  );

  return getWeatherIcon(
    selectedCode,
    isDay
  );
};



const thermoIconUrl = new URL('/node_modules/@meteocons/svg/fill/thermometer.svg', import.meta.url).href;
const sunriseIconUrl = new URL('/node_modules/@meteocons/svg/fill/sunrise.svg', import.meta.url).href;
const sunsetIconUrl = new URL('/node_modules/@meteocons/svg/fill/sunset.svg', import.meta.url).href;


const hourlyData = computed(() => {
  if (!weather.value) return [];

  const currentLocalTime = getLocalTimeForTimezone(
    weather.value.timezone
  );

  const currentHourStr = currentLocalTime.slice(0, 13) + ':00';

  const startIndex = Math.max(
    0,
    weather.value.hourly.time.findIndex(
      t => t === currentHourStr
    )
  );

  const sunEvents = [];
  if (weather.value.daily) {
    weather.value.daily.sunrise.forEach(time => sunEvents.push({ type: 'sunrise', fullTime: time, icon: sunriseIconUrl, label: 'Aufgang' }));
    weather.value.daily.sunset.forEach(time => sunEvents.push({ type: 'sunset', fullTime: time, icon: sunsetIconUrl, label: 'Untergang' }));
  }

  const list = [];

  for (let i = 0; i < 25; i++) {
    const idx = startIndex + i;
    const currentStr = weather.value.hourly.time[idx];
    if (!currentStr) break;
    const nextStr = weather.value.hourly.time[idx + 1];

    list.push({
      type: 'weather',
      time: currentStr.slice(11, 16),
      temp: Math.round(weather.value.hourly.temperature_2m[idx]),
      prob: weather.value.hourly.precipitation_probability[idx],
      precipitation: weather.value.hourly.precipitation[idx],
      icon: getWeatherIcon(
        weather.value.hourly.weather_code[idx],
        weather.value.hourly.is_day[idx],
        weather.value.hourly.precipitation[idx]

      ),

      description: weatherDescriptions[weather.value.hourly.weather_code[idx]] || "Aktuell"
    });

    if (nextStr) {
      sunEvents.forEach(event => {
        if (event.fullTime >= currentStr && event.fullTime < nextStr) {
          list.push({
            type: event.type,
            time: event.fullTime.slice(11, 16),
            label: event.label,
            icon: event.icon
          });
        }
      });
    }
  }
  return list;
});

const dailyData = computed(() => {
  if (!weather.value?.daily || !weather.value?.hourly) {
    return [];
  }

  const daily = weather.value.daily;
  const hourly = weather.value.hourly;

  return daily.time.slice(0, 8).map((date, dayIndex) => {

    const dayIndices = [];

    hourly.time.forEach((time, index) => {
      if (
        time.startsWith(date) &&
        hourly.is_day[index] === 1
      ) {
        dayIndices.push(index);
      }
    });

    const nightIndices = [];

    const sunset = daily.sunset[dayIndex];
    const nextSunrise = daily.sunrise[dayIndex + 1];

    if (sunset && nextSunrise) {

      hourly.time.forEach((time, index) => {

        if (
          time >= sunset &&
          time < nextSunrise &&
          hourly.is_day[index] === 0
        ) {
          nightIndices.push(index);
        }

      });

    }

    const dayIcon = getIconForPeriod(dayIndices, 1);

    const nightIcon = getIconForPeriod(
      nightIndices,
      0
    );


    const dayName = new Date(`${date}T12:00:00`).toLocaleDateString(
      'de-DE',
      {
        weekday: 'short'
      }
    );


    const precipitation = daily.precipitation_sum[dayIndex] ?? 0;


    return {
      date,
      day: dayName,

      tempMax: Math.round(
        daily.temperature_2m_max[dayIndex]
      ),

      tempMin: Math.round(
        daily.temperature_2m_min[dayIndex]
      ),

      precipitation,

      dayIcon,
      nightIcon
    };
  });
});



</script>

<template>
  <div class="weather-app">


    <div class="search-container">

      <div class="search-row">

        <input v-model="searchText" @keyup="searchCity" type="text" class="city-search" placeholder="Stadt suchen..." />

        <button class="location-button" @click="getCurrentLocation" title="Aktuellen Standort verwenden"
          aria-label="Aktuellen Standort verwenden">
          📍
        </button>

      </div>

      <div v-if="searchResults.length > 0" class="search-results">

        <div v-for="city in searchResults" :key="city.id" class="search-result" @click="selectSearchCity(city)">
          <strong>{{ city.name }}</strong>

          <span>
            {{ city.country }}

            <template v-if="city.admin1">
              · {{ city.admin1 }}
            </template>
          </span>
        </div>

      </div>

    </div>


    <div v-if="weather && hourlyData.length > 0" class="current-weather-card">
      <div class="current-info">
        <h1 style="font-weight: bold;">{{ selectedCity.name }}</h1>
        <p class="current-weather">{{ hourlyData[0]?.description }}</p>
      </div>
      <div class="current-details">
        <span class="current-temp" style="font-weight: bold;">
          <img :src="hourlyData[0].icon" alt=" " class="current-icon"
            style="height: 75px; width: 75px; margin-bottom: -25px;" />
          {{ hourlyData[0].temp }}{{ weather.hourly_units.temperature_2m }}
        </span>
      </div>
      <span class="current-apparent-temperature">Gefühlt wie {{ Math.round(weather.current.apparent_temperature) }}{{
        weather.hourly_units.temperature_2m }}</span>
    </div>
    <h2 class="forecast-title">24-Stunden-Vorhersage</h2>

    <div v-if="weather" class="weather-box">
      <div v-for="(hour, index) in hourlyData" :key="index" class="hour-box" :class="hour.type">

        <!-- Wenn es eine normale Wetterstunde ist -->
        <template v-if="hour.type === 'weather'">
          <span class="time">{{ hour.time }}</span>
          <img :src="hour.icon" alt="Wetter Icon"
            style="width: 50px; height: 50px; margin: 5px 0; margin-top: -5px; margin-bottom: -5px;" />
          <span class="temperature-box">
            <img :src="thermoIconUrl" alt=" " class="thermo-icon" />
            {{ hour.temp }}{{ weather.hourly_units.temperature_2m }}
          </span>
          <span class="prob-box">
            💧 {{ hour.precipitation }}{{ weather.hourly_units.precipitation }}
          </span>

        </template>

        <template v-else>
          <span class="time">{{ hour.time }}</span>
          <img :src="hour.icon" :alt="hour.label"
            style="width: 50px; height: 50px; margin: 5px 0; margin-top: -5px; margin-bottom: -5px;" />

          <span class="sun-label">{{ hour.label }}</span>
          <span class="prob-box" style="visibility: hidden;">💧 0%</span>

        </template>

      </div>
    </div>

    <h2 class="forecast-title">7-Tage-Vorhersage</h2>

    <div v-if="weather" class="daily-weather-box">

      <div v-for="(day, index) in dailyData" :key="day.date" class="hour-box daily-box">

        <span class="time">
          {{ index === 0 ? 'Heute' : `${day.day} ${new Date(day.date + 'T12:00:00').toLocaleDateString('de-DE', {
            day: '2-digit',
            month: '2-digit'
          })}` }}
        </span>


        <div class="daily-icons">

          <img :src="day.dayIcon" alt="Tag" class="daily-icon" />

          <img :src="day.nightIcon" alt="Nacht" class="daily-icon" />

        </div>


        <span class="temperature-box daily-temperature">
          <span class="temp-max">
            {{ day.tempMax }}{{ weather.daily_units.temperature_2m_max }}
          </span>

          <span class="temp-separator">/</span>

          <span class="temp-min">
            {{ day.tempMin }}{{ weather.daily_units.temperature_2m_min }}
          </span>
        </span>

        <span class="prob-box">
          💧 {{ day.precipitation }}{{ weather.daily_units.precipitation_sum }}
        </span>

      </div>

    </div>



  </div>

</template>

<style scoped>
.weather-app {
  width: 100%;
  max-width: 600px;
  margin: 20px auto;
  display: flex;
  flex-direction: column;
  gap: 15px;
  font-family: system-ui, -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
}

#cities {
  width: 180px;
  height: 38px;
  padding: 0 10px;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  background-color: #ffffff;
  color: #334155;
  font-weight: 500;
  cursor: pointer;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.05);
  outline: none;
  transition: border-color 0.2s, box-shadow 0.2s;
}

#cities:focus {
  border-color: #3b82f6;
  box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.15);
}

.weather-box {
  width: 100%;
  display: flex;
  gap: 8px;
  padding: 10px 4px;
  overflow-x: auto;
  scroll-behavior: smooth;
  -webkit-overflow-scrolling: touch;
}

.hour-box {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: space-between;
  text-align: center;
  padding: 14px 10px;
  background: #ffffff;
  border: 1px solid #f1f5f9;
  border-radius: 14px;
  min-width: 85px;
  box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.05), 0 2px 4px -1px rgba(0, 0, 0, 0.03);
  transition: transform 0.2s, box-shadow 0.2s;
}

.hour-box:hover {
  transform: translateY(-2px);
  box-shadow: 0 10px 15px -3px rgba(0, 0, 0, 0.08);
}

/* Texte und Werte */
.time {
  font-size: 0.85rem;
  font-weight: 600;
  color: #64748b;
}

.temperature-box {
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 1.05rem;
  font-weight: 700;
  color: #1e293b;
  height: 32px;
}

.thermo-icon {
  width: 24px;
  height: 24px;
  margin-right: -4px;
}

.prob-box {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 3px;
  font-size: 0.75rem;
  font-weight: 600;
  color: #0284c7;
  white-space: nowrap;
}

.sunrise,
.sunset {
  border: 1px solid #fef08a;
  box-shadow: 0 4px 6px -1px rgba(234, 179, 8, 0.05);
}

.sunrise {
  background: linear-gradient(180deg, #fffdf5 0%, #fefce8 100%);
}

.sunset {
  background: linear-gradient(180deg, #fffaf8 0%, #ffedd5 100%);
}

.sun-label {
  display: flex;
  align-items: center;
  justify-content: center;
  height: 32px;
  font-size: 0.8rem;
  font-weight: 700;
}

.sunrise .sun-label {
  color: #d97706;
}

.sunset .sun-label {
  color: #ea580c;
}

.current-temp {
  font-size: xx-large;
}

.daily-weather-box {
  width: 100%;
  display: flex;
  gap: 8px;
  padding: 10px 4px;
  overflow-x: auto;
  scroll-behavior: smooth;
  -webkit-overflow-scrolling: touch;
}

.daily-box {
  min-width: 100px;
}

.daily-temperature {
  gap: 3px;
}

.temp-max {
  color: #1e293b;
  font-weight: 700;
}

.temp-separator {
  color: #94a3b8;
  font-weight: 400;
}

.temp-min {
  color: #64748b;
  font-weight: 600;
}

.forecast-title {
  margin: 5px 4px 0;
  padding-bottom: 6px;
  font-size: 1.1rem;
  font-weight: 700;
  color: #1e293b;
  border-bottom: 1px solid #e2e8f0;
}

.city-search {
  width: 100%;
  height: 40px;
  padding: 0 12px;
  box-sizing: border-box;

  border: 1px solid #e2e8f0;
  border-radius: 8px;

  font-size: 1rem;
  outline: none;
}

.city-search:focus {
  border-color: #3b82f6;
}

.search-container {
  position: relative;
  width: 100%;
}

.search-results {
  position: absolute;
  top: calc(100% + 5px);
  left: 0;
  right: 0;

  z-index: 1000;

  background: white;
  border: 1px solid #e2e8f0;
  border-radius: 8px;

  overflow: hidden;

  box-shadow: 0 8px 20px rgba(0, 0, 0, 0.12);
}

.search-result {
  display: flex;
  flex-direction: column;
  gap: 3px;

  padding: 10px 12px;

  cursor: pointer;
  border-bottom: 1px solid #f1f5f9;
}

.search-result:last-child {
  border-bottom: none;
}

.search-result:hover {
  background: #f8fafc;
}

.search-result strong {
  color: #1e293b;
}

.search-result span {
  font-size: 0.8rem;
  color: #64748b;
}

.daily-icons {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 2px;
  height: 55px;
}

.daily-icon {
  width: 42px;
  height: 42px;
}

.search-row {
  display: flex;
  gap: 8px;
  width: 100%;
}

.city-search {
  flex: 1;
}

.location-button {
  width: 42px;
  height: 40px;
  flex-shrink: 0;

  display: flex;
  align-items: center;
  justify-content: center;

  border: 1px solid #e2e8f0;
  border-radius: 8px;

  background: #ffffff;
  color: #334155;

  font-size: 1.1rem;
  cursor: pointer;

  transition:
    background-color 0.2s,
    border-color 0.2s,
    transform 0.1s;
}

.location-button:hover {
  background: #f8fafc;
  border-color: #3b82f6;
}

.location-button:active {
  transform: scale(0.95);
}

.current-apparent-temperature {
  font-size: medium;
}
</style>