<script setup>
import { ref, watch, computed } from 'vue'

const currentHour = new Date().toLocaleString('sv').slice(0, 14) + '00';

const searchText = ref('')
const searchResults = ref([])
const selectedCity = ref(null);
const weather = ref(null);

// Menü-Zustand & Animation
const isMenuOpen = ref(false);
const isMenuClosing = ref(false);
const activeMenuIndex = ref(null);

// Gespeicherte Städte laden
const savedCities = ref(JSON.parse(localStorage.getItem('weather_saved_cities') || '[]'));

// Aktuelle "Standardstadt" (Standort oder Fallback)
const currentStandardCity = ref(JSON.parse(localStorage.getItem('weather_standard_city') || 'null'));

// Menü kontrolliert schließen mit Animation
const closeMenu = () => {
  isMenuClosing.value = true;
  setTimeout(() => {
    isMenuOpen.value = false;
    isMenuClosing.value = false;
    activeMenuIndex.value = null;
  }, 200); // 200ms entspricht der Animationsdauer
};

// Hilfsfunktion: Setzt die aktive Stadt und speichert sie als "zuletzt geöffnet"
const setAndSaveCurrentCity = (cityObj) => {
  selectedCity.value = cityObj;
  localStorage.setItem('weather_last_opened_city', JSON.stringify(cityObj));
};

// Stadt zu Favoriten hinzufügen (über das kleine + im Menü)
const addCityToSaved = (city, event) => {
  event.stopPropagation();
  const exists = savedCities.value.some(
    c => Math.abs(c.lan - city.lan) < 0.01 && Math.abs(c.lon - city.lon) < 0.01
  );
  if (!exists) {
    savedCities.value.unshift({ name: city.name, lan: city.lan, lon: city.lon });
    if (savedCities.value.length > 20) savedCities.value.pop();
    localStorage.setItem('weather_saved_cities', JSON.stringify(savedCities.value));
  }
};

// Gespeicherte Stadt löschen
const removeSavedCity = (index, event) => {
  event.stopPropagation();
  const removed = savedCities.value[index];
  savedCities.value.splice(index, 1);
  localStorage.setItem('weather_saved_cities', JSON.stringify(savedCities.value));
  activeMenuIndex.value = null;

  // Falls die gelöschte Stadt gerade aktiv war, auf Standardstadt zurückfallen
  if (selectedCity.value && !selectedCity.value.isPin && selectedCity.value.name === removed.name) {
    if (currentStandardCity.value) {
      setAndSaveCurrentCity({ ...currentStandardCity.value, isPin: true });
    }
  }
};

// Aus gespeicherten Städten auswählen
const selectSavedCity = (city) => {
  setAndSaveCurrentCity({ name: city.name, lan: city.lan, lon: city.lon, isPin: false });
  closeMenu();
};

// Standardstadt (Pin) auswählen
const selectStandardCity = () => {
  if (currentStandardCity.value) {
    setAndSaveCurrentCity({ ...currentStandardCity.value, isPin: true });
    closeMenu();
  }
};

// Suchergebnis anklicken -> lädt als Standardstadt/aktuelle Stadt
const selectSearchCity = (city) => {
  const chosenCity = {
    name: city.name,
    lan: city.latitude,
    lon: city.longitude,
    isPin: true
  };

  currentStandardCity.value = chosenCity;
  localStorage.setItem('weather_standard_city', JSON.stringify(chosenCity));

  setAndSaveCurrentCity(chosenCity);

  searchText.value = "";
  searchResults.value = [];
  closeMenu();
}

// Standort ermitteln (auf Knopfdruck oder beim Start)
// Standort ermitteln (auf Knopfdruck oder beim Start)
const fetchGeolocation = (isStartup = false) => {

  if (!navigator.geolocation) {
    if (!isStartup) alert('Dein Browser unterstützt keine Standortbestimmung.');
    if (!selectedCity.value) loadFallbackCity();
    return;
  }

  // Sicherheits-Timeout, falls der Browser komplett blockiert und weder success noch error feuert
  let locationResolved = false;
  const safetyTimer = setTimeout(() => {
    if (!locationResolved) {
      if (!isStartup) {
        alert('Standort-Abfrage dauerhaft blockiert oder fehlgeschlagen. Bitte prüfe die Browsereinstellungen.');
      }
      if (!selectedCity.value) loadFallbackCity();
    }
  }, 8000);

  navigator.geolocation.getCurrentPosition(
    async (position) => {
      locationResolved = true;
      clearTimeout(safetyTimer);
      localStorage.setItem('weather_loc_permission', 'granted');

      const latitude = position.coords.latitude;
      const longitude = position.coords.longitude;

      let locationName = 'Mein Standort';
      try {
        const response = await fetch(
          `https://nominatim.openstreetmap.org/reverse?lat=${latitude}&lon=${longitude}&format=jsonv2&zoom=10&addressdetails=1&accept-language=de`
        );
        const data = await response.json();
        const address = data.address || {};
        locationName = address.city || address.town || address.village || address.municipality || 'Mein Standort';
      } catch (e) {
        console.error("Reverse Geocoding Fehler:", e);
      }

      const newStdCity = { name: locationName, lan: latitude, lon: longitude, isPin: true };
      currentStandardCity.value = newStdCity;
      localStorage.setItem('weather_standard_city', JSON.stringify(newStdCity));

      if (isStartup && !localStorage.getItem('weather_last_opened_city')) {
        setAndSaveCurrentCity(newStdCity);
      } else if (!isStartup) {
        // Wenn manuell geklickt, direkt dorthin wechseln!
        setAndSaveCurrentCity(newStdCity);
      }
    },
    (error) => {
      locationResolved = true;
      clearTimeout(safetyTimer);
      localStorage.setItem('weather_loc_permission', 'denied');
      console.error("Geolocation Fehler-Code:", error.code, error.message);
      
      if (!isStartup) {
        alert('Standort konnte nicht ermittelt werden. Fehler: ' + error.message);
      }
      if (!selectedCity.value) loadFallbackCity();
    },
    { enableHighAccuracy: true, timeout: 7000, maximumAge: 0 }
  );
};

// Fallback Logik (IP-Hauptstadt oder erste gespeicherte Stadt)
const loadFallbackCity = async () => {
  // 1. Wenn es bereits eine gespeicherte "Zuletzt geöffnet"-Stadt gibt, nimm diese
  const lastOpened = localStorage.getItem('weather_last_opened_city');
  if (lastOpened) {
    selectedCity.value = JSON.parse(lastOpened);
    return;
  }

  // 2. Ansonsten Standardstadt
  if (currentStandardCity.value) {
    setAndSaveCurrentCity({ ...currentStandardCity.value, isPin: true });
    return;
  }

  // 3. Ansonsten erster Favorit
  if (savedCities.value.length > 0) {
    setAndSaveCurrentCity({ ...savedCities.value[0], isPin: false });
    return;
  }

  // 4. IP-basierte Landeshauptstadt als Fallback
  try {
    const res = await fetch('https://ipapi.co/json/');
    const data = await res.json();
    if (data && data.city) {
      const geoRes = await fetch(`https://geocoding-api.open-meteo.com/v1/search?name=${encodeURIComponent(data.city)}&count=1&language=de&format=json`);
      const geoData = await geoRes.json();
      if (geoData.results && geoData.results.length > 0) {
        const fallback = { name: geoData.results[0].name, lan: geoData.results[0].latitude, lon: geoData.results[0].longitude, isPin: true };
        currentStandardCity.value = fallback;
        localStorage.setItem('weather_standard_city', JSON.stringify(fallback));
        setAndSaveCurrentCity(fallback);
        return;
      }
    }
    throw new Error();
  } catch (e) {
    const fallback = { name: "Berlin", lan: 52.52437, lon: 13.41053, isPin: true };
    currentStandardCity.value = fallback;
    setAndSaveCurrentCity(fallback);
  }
};

// Initialisierung beim Start
const initApp = () => {
  const lastOpened = localStorage.getItem('weather_last_opened_city');
  const storedStd = localStorage.getItem('weather_standard_city');
  const savedPerm = localStorage.getItem('weather_loc_permission');

  if (storedStd) currentStandardCity.value = JSON.parse(storedStd);

  if (lastOpened) {
    selectedCity.value = JSON.parse(lastOpened);
  } else {
    loadFallbackCity();
  }

  // Im Hintergrund Standort aktualisieren, falls vorher erlaubt
  if (savedPerm === 'granted') {
    fetchGeolocation(true);
  }
};

initApp();


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
  // Klar / Wolken (Standard WMO 0, 1, 2, 3)
  0: { day: 'clear-day', night: 'clear-night' },
  1: { day: 'mostly-clear-day', night: 'mostly-clear-night' },
  2: { day: 'partly-cloudy-day', night: 'partly-cloudy-night' },
  3: { day: 'overcast-day', night: 'overcast-night' },

  // Nebel (WMO 45, 48) - Fallback, wird unten dynamisch über cloud_cover verfeinert
  45: { day: 'fog-day', night: 'fog-night' },
  48: { day: 'fog-day', night: 'fog-night' },

  // Nieselregen / Leichter Regen (WMO 51-57, 61, 80) -> Stufe 1 (Drizzle)
  51: { day: 'mostly-clear-day-drizzle', night: 'mostly-clear-night-drizzle' },
  53: { day: 'mostly-clear-day-drizzle', night: 'mostly-clear-night-drizzle' },
  55: { day: 'mostly-clear-day-drizzle', night: 'mostly-clear-night-drizzle' },
  56: { day: 'mostly-clear-day-drizzle', night: 'mostly-clear-night-drizzle' },
  57: { day: 'mostly-clear-day-drizzle', night: 'mostly-clear-night-drizzle' },
  61: { day: 'mostly-clear-day-rain', night: 'mostly-clear-night-rain' },
  80: { day: 'mostly-clear-day-rain', night: 'mostly-clear-night-rain' },

  // Mäßiger Regen (WMO 63, 66, 81) -> Stufe 3 (Partly Cloudy Rain)
  63: { day: 'partly-cloudy-day-rain', night: 'partly-cloudy-night-rain' },
  66: { day: 'partly-cloudy-day-rain', night: 'partly-cloudy-night-rain' },
  81: { day: 'partly-cloudy-day-rain', night: 'partly-cloudy-night-rain' },

  // Starker Regen (WMO 65, 67, 82) -> Stufe 4 (Overcast Rain)
  65: { day: 'overcast-day-rain', night: 'overcast-night-rain' },
  67: { day: 'overcast-day-rain', night: 'overcast-night-rain' },
  82: { day: 'overcast-day-rain', night: 'overcast-night-rain' },

  // Schnee (aufsteigend nach Bewölkung)
  71: { day: 'mostly-clear-day-snow', night: 'mostly-clear-night-snow' },
  73: { day: 'partly-cloudy-day-snow', night: 'partly-cloudy-night-snow' },
  75: { day: 'overcast-day-snow', night: 'overcast-night-snow' },
  77: { day: 'overcast-day-snow', night: 'overcast-night-snow' },
  85: { day: 'partly-cloudy-day-snow', night: 'partly-cloudy-night-snow' },
  86: { day: 'extreme-day-snow', night: 'extreme-night-snow' },

  // Gewitter OHNE Niederschlag (WMO 95, 97 - rein nach Bewölkungsstufe)
  95: { day: 'thunderstorms-day', night: 'thunderstorms-night' },
  97: { day: 'thunderstorms-extreme-day', night: 'thunderstorms-extreme-night' },

  // Gewitter MIT leichtem Hagel/Regen (WMO 96, 99)
  96: { day: 'thunderstorms-day-hail', night: 'thunderstorms-night-hail' },
  99: { day: 'extreme-thunderstorms-day-hail', night: 'extreme-thunderstorms-night-hail' }
};

const getWeatherIcon = (wmoCode, isDay, precipitation = 0, cloudCover = 50) => {
  let iconName = '';

  // 1. NEBEL-BEHANDLUNG (WMO 45, 48) anhand der echten Wolkenmenge (cloud_cover in %)
  if ([45, 48].includes(wmoCode)) {
    if (cloudCover < 20) {
      iconName = isDay === 1 ? 'mostly-clear-day-fog' : 'mostly-clear-night-fog';
    } else if (cloudCover < 70) {
      iconName = isDay === 1 ? 'partly-cloudy-day-fog' : 'partly-cloudy-night-fog';
    } else {
      iconName = isDay === 1 ? 'overcast-day-fog' : 'overcast-night-fog';
    }
  } 
  // 2. GEWITTER-BEHANDLUNG (WMO 95, 96, 97, 99)
  else if ([95, 96, 97, 99].includes(wmoCode)) {
    const hasRain = precipitation > 0;
    
    if (hasRain) {
      if (precipitation < 1.0) {
        iconName = isDay === 1 ? 'thunderstorms-mostly-clear-day-drizzle' : 'thunderstorms-mostly-clear-night-drizzle';
      } else if (precipitation < 3.0) {
        iconName = isDay === 1 ? 'thunderstorms-day-rain' : 'thunderstorms-night-rain';
      } else {
        iconName = isDay === 1 ? 'thunderstorms-overcast-day-rain' : 'thunderstorms-overcast-night-rain';
      }
    } else {
      if (cloudCover < 30) {
        iconName = isDay === 1 ? 'thunderstorms-mostly-clear-day' : 'thunderstorms-mostly-clear-night';
      } else if (cloudCover < 70) {
        iconName = isDay === 1 ? 'thunderstorms-day' : 'thunderstorms-night';
      } else {
        iconName = isDay === 1 ? 'thunderstorms-overcast-day' : 'thunderstorms-overcast-night';
      }
    }
  } 
  // 3. STANDARD- ODER REGEN-/SCHNEE-BEHANDLUNG
  else {
    const icon = meteoconIcons[wmoCode];
    if (!icon) {
      iconName = isDay === 1 ? 'mostly-clear-day' : 'mostly-clear-night';
    } else {
      iconName = isDay === 1 ? icon.day : icon.night;
    }
  }

  // Direkt die CDN-URL zurückgeben ohne new URL()
  return `https://cdn.jsdelivr.net/npm/@meteocons/svg/fill/${iconName}.svg`;
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
  let lastSeenDay = null; // Hilfsvariable, um den Tageswechsel zu erkennen

  // 48 Stunden (49 Indizes mit Startstunde)
  for (let i = 0; i < 49; i++) {
    const idx = startIndex + i;
    const currentStr = weather.value.hourly.time[idx];
    if (!currentStr) break;
    const nextStr = weather.value.hourly.time[idx + 1];

    const datePart = currentStr.slice(0, 10); // Format: "YYYY-MM-DD"
    const timePart = currentStr.slice(11, 16); // Format: "HH:MM"

    // Prüfen, ob ein neuer Tag beginnt
    let dayLabel = "";
    if (datePart !== lastSeenDay) {
      lastSeenDay = datePart;
      
      const todayStr = currentLocalTime.slice(0, 10);
      const tomorrowStr = new Date(new Date(todayStr).setDate(new Date(todayStr).getDate() + 1)).toISOString().slice(0, 10);

      if (datePart === todayStr) {
        dayLabel = 'HEUTE';
      } else if (datePart === tomorrowStr) {
        dayLabel = 'MORGEN';
      } else {
        dayLabel = new Date(`${datePart}T12:00:00`).toLocaleDateString('de-DE', { weekday: 'short' }).toUpperCase();
      }
    }

    list.push({
      type: 'weather',
      time: timePart,
      dayLabel: dayLabel, // Wird nur gesetzt, wenn sich der Tag ändert!
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
  <div class="weather-app" @click="activeMenuIndex = null">

    <!-- OBERE LEISTE: Nur Menü-Symbol und direkt daneben der aktuelle Stadtname -->
    <div class="app-top-bar">
      <button class="menu-icon-btn" @click.stop="isMenuOpen = true" aria-label="Menü öffnen">
        ☰
      </button>
      <div class="current-bar-title">
        <h2 style="font-weight: bold;">{{ selectedCity?.name || 'Lade...' }}</h2>
      </div>
    </div>

    <!-- AUFKLAPPMENÜ (Mobile-optimiert mit Slide-Animation) -->
    <div v-if="isMenuOpen" class="side-menu-overlay" :class="{ 'closing': isMenuClosing }" @click.self="closeMenu">
      <div class="side-menu" :class="{ 'closing': isMenuClosing }">

        <!-- Suchleiste oben im Menü mit Standort-Button -->
        <div class="search-container">
          <div class="search-row">
            <input v-model="searchText" @keyup="searchCity" type="text" class="city-search"
              placeholder="Stadt suchen..." />
            <button class="location-button" @click="fetchGeolocation(false); closeMenu();"
              title="Aktuellen Standort ermitteln">
              📍
            </button>
          </div>

          <div v-if="searchResults.length > 0" class="search-results">
            <div v-for="city in searchResults" :key="city.id" class="search-result" @click="selectSearchCity(city)">
              <strong>{{ city.name }}</strong>
              <span>{{ city.country }} <template v-if="city.admin1">· {{ city.admin1 }}</template></span>
            </div>
          </div>

        </div>

        <div class="menu-separator"></div>

        <!-- Standardstadt (Standort / Fallback) -->
        <div v-if="currentStandardCity" class="menu-city-row standard-city-row" @click="selectStandardCity">
          <div class="city-row-content">
            <span class="city-name">{{ currentStandardCity.name }}</span>
          </div>
          <button class="add-quick-btn" @click="addCityToSaved(currentStandardCity, $event)"
            title="Zu Liste hinzufügen">+</button>
        </div>

        <div class="menu-separator"></div>

        <!-- Gespeicherte Städte Liste -->
        <div class="saved-cities-container">
          <div v-if="savedCities.length === 0" class="no-cities-hint">Keine gespeicherten Städte</div>

          <div v-for="(city, index) in savedCities" :key="index" class="menu-city-row" @click="selectSavedCity(city)">
            <span class="city-name">{{ city.name }}</span>

            <!-- Dezentes 3-Punkt-Menü -->
            <div class="dropdown-wrapper" @click.stop>
              <button class="dots-btn" @click="activeMenuIndex = activeMenuIndex === index ? null : index">⋮</button>
              <div v-if="activeMenuIndex === index" class="dropdown-popup">
                <button @click="removeSavedCity(index, $event)">Löschen</button>
              </div>
            </div>
          </div>
        </div>

      </div>
    </div>

    <div v-if="weather && hourlyData.length > 0" class="current-weather-card">
      
      <div class="current-details">
        <span class="current-temp" style="font-weight: bold;">
          <img :src="hourlyData[0].icon" alt=" " class="current-icon"
            style="height: 75px; width: 75px; margin-bottom: -25px;" />
          {{ hourlyData[0].temp }}{{ weather.hourly_units.temperature_2m }}
        </span>
      </div>
      <div class="current-info">
        <p class="current-weather">{{ hourlyData[0]?.description }}</p>
      </div>
      <br>
      <span class="current-apparent-temperature">Gefühlt wie {{ Math.round(weather.current.apparent_temperature)
      }}{{
          weather.hourly_units.temperature_2m }}</span>
    </div>
    <h2 class="forecast-title">24-Stunden-Vorhersage</h2>

    <div v-if="weather" class="weather-box">
      <div v-for="(hour, index) in hourlyData" :key="index" class="hour-box" :class="hour.type">

        <!-- Wenn es eine normale Wetterstunde ist -->
        <template v-if="hour.type === 'weather'">
          <div class="badge-wrapper">
            <span v-if="hour.dayLabel" class="day-badge">{{ hour.dayLabel }}</span>
          </div>
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
          <div class="badge-wrapper">
            <span v-if="hour.dayLabel" class="day-badge">{{ hour.dayLabel }}</span>
          </div>
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
  margin: 0px auto;
  display: flex;
  flex-direction: column;
  gap: 15px;
  font-family: system-ui, -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
  color: #1e293b;
}

/* =========================================================
   STUNDENVORHERSAGE
   ========================================================= */
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
  transition: transform 0.2s, box-shadow 0.2s, background-color 0.2s, border-color 0.2s;
}

.hour-box:hover {
  transform: translateY(-2px);
  box-shadow: 0 10px 15px -3px rgba(0, 0, 0, 0.08);
}

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

/* =========================================================
   SONNENAUFGANG / SONNENUNTERGANG
   ========================================================= */
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

.current-apparent-temperature {
  font-size: medium;
}

/* =========================================================
   7-TAGE-VORHERSAGE
   ========================================================= */
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

.forecast-title {
  margin: 5px 4px 0;
  padding-bottom: 6px;
  font-size: 1.1rem;
  font-weight: 700;
  color: #1e293b;
  border-bottom: 1px solid #e2e8f0;
}

/* =========================================================
   SUCHE & MENÜ (Lightmode)
   ========================================================= */
.search-container {
  position: relative;
  width: 100%;
}

.search-row {
  display: flex;
  gap: 8px;
  width: 100%;
}

.city-search {
  flex: 1;
  width: 100%;
  height: 40px;
  padding: 0 12px;
  box-sizing: border-box;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  background-color: #ffffff;
  color: #1e293b;
  font-size: 1rem;
  outline: none;
  transition: border-color 0.2s, background-color 0.2s, color 0.2s, box-shadow 0.2s;
}

.city-search::placeholder {
  color: #94a3b8;
}

.city-search:focus {
  border-color: #3b82f6;
  box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.12);
}

.search-results {
  position: absolute;
  top: calc(100% + 5px);
  left: 0;
  right: 0;
  z-index: 1000;
  background: #ffffff;
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
  transition: background-color 0.15s, color 0.15s;
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
  transition: background-color 0.2s, border-color 0.2s, color 0.2s, transform 0.1s;
}

.location-button:hover {
  background: #f8fafc;
  border-color: #3b82f6;
}

.location-button:active {
  transform: scale(0.95);
}

.app-top-bar {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 4px 4px;
}

.menu-icon-btn {
  background: none;
  border: none;
  font-size: 1.5rem;
  cursor: pointer;
  color: inherit;
  padding: 4px 2px;
  border-radius: 6px;
}

.menu-icon-btn:hover {
  background: rgba(0, 0, 0, 0.05);
}

.current-bar-title {
  font-size: 1.3rem;
  font-weight: 700;
  display: flex;
  align-items: center;
  gap: 6px;
}

/* Overlay & Slide-In Animation von links */
.side-menu-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  background: rgba(0, 0, 0, 0.4);
  z-index: 3000;
  display: flex;
  justify-content: flex-start;
  animation: fadeInOverlay 0.25s ease forwards;
}

.side-menu-overlay.closing {
  animation: fadeOutOverlay 0.2s ease forwards;
}

.side-menu {
  width: 85%;
  max-width: 340px;
  height: 100%;
  background: #ffffff;
  box-shadow: 4px 0 20px rgba(0, 0, 0, 0.15);
  padding: 20px 15px;
  box-sizing: border-box;
  display: flex;
  flex-direction: column;
  gap: 12px;
  overflow-y: auto;
  animation: slideInLeft 0.25s cubic-bezier(0.16, 1, 0.3, 1) forwards;
}

.side-menu.closing {
  animation: slideOutLeft 0.2s cubic-bezier(0.16, 1, 0.3, 1) forwards;
}

@keyframes fadeInOverlay {
  from { background: rgba(0, 0, 0, 0); }
  to { background: rgba(0, 0, 0, 0.4); }
}

@keyframes fadeOutOverlay {
  from { background: rgba(0, 0, 0, 0.4); }
  to { background: rgba(0, 0, 0, 0); }
}

@keyframes slideInLeft {
  from { transform: translateX(-100%); }
  to { transform: translateX(0); }
}

@keyframes slideOutLeft {
  from { transform: translateX(0); }
  to { transform: translateX(-100%); }
}

.menu-separator {
  height: 1px;
  background-color: #e2e8f0;
  width: 100%;
  margin: 4px 0;
}

.add-quick-btn {
  background: #f1f5f9;
  border: 1px solid #cbd5e1;
  color: #334155;
  width: 28px;
  height: 28px;
  border-radius: 50%;
  font-weight: bold;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
}

.add-quick-btn:hover {
  background: #3b82f6;
  color: white;
  border-color: #3b82f6;
}

.menu-city-row {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 12px 10px;
  border-radius: 8px;
  cursor: pointer;
  transition: background 0.15s;
}

.menu-city-row:hover {
  background: #f8fafc;
}

.city-row-content {
  display: flex;
  align-items: center;
  gap: 8px;
  flex: 1;
}

.city-name {
  font-weight: 600;
  color: #1e293b;
  font-size: 0.95rem;
}

.saved-cities-container {
  display: flex;
  flex-direction: column;
  gap: 4px;
  overflow-y: auto;
}

.no-cities-hint {
  font-size: 0.85rem;
  color: #94a3b8;
  text-align: center;
  padding: 10px;
  font-style: italic;
}

.dropdown-wrapper {
  position: relative;
}

.dots-btn {
  background: none;
  border: none;
  font-size: 1.2rem;
  cursor: pointer;
  color: #64748b;
  padding: 0 8px;
}

.dropdown-popup {
  position: absolute;
  right: 0;
  top: 100%;
  background: white;
  border: 1px solid #e2e8f0;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
  border-radius: 6px;
  z-index: 3100;
  overflow: hidden;
}

.dropdown-popup button {
  background: none;
  border: none;
  padding: 8px 16px;
  text-align: left;
  width: 100%;
  cursor: pointer;
  font-size: 0.85rem;
  color: #ef4444;
  font-weight: 600;
}

.dropdown-popup button:hover {
  background: #fef2f2;
}

.badge-wrapper {
  height: 10px;
  display: flex;
  align-items: center;
  justify-content: center;
  margin-bottom: 2px;
}

.day-badge {
  font-size: 0.6rem;
  font-weight: 700;
  background: #e0f2fe;
  color: #0369a1;
  padding: 3px 5px;
  border-radius: 4px;
  text-transform: uppercase;
  letter-spacing: 0.5px;
  line-height: 1.2;
}


/* =========================================================
   AUTOMATISCHER DARKMODE
   ========================================================= */
@media (prefers-color-scheme: dark) {
  .weather-app {
    color: #e5e7eb;
  }

  .hour-box {
    background: #111827;
    border-color: #1f2937;
    box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.25), 0 2px 4px -1px rgba(0, 0, 0, 0.18);
  }

  .hour-box:hover {
    box-shadow: 0 10px 15px -3px rgba(0, 0, 0, 0.35);
  }

  .time {
    color: #9ca3af;
  }

  .temperature-box {
    color: #f3f4f6;
  }

  .prob-box {
    color: #38bdf8;
  }

  .sunrise {
    background: linear-gradient(180deg, #292615 0%, #302d16 100%);
    border-color: #665d22;
  }

  .sunset {
    background: linear-gradient(180deg, #2d211d 0%, #34241c 100%);
    border-color: #704027;
  }

  .sunrise .sun-label {
    color: #fbbf24;
  }

  .sunset .sun-label {
    color: #fb923c;
  }

  .temp-max {
    color: #f3f4f6;
  }

  .temp-separator {
    color: #6b7280;
  }

  .temp-min {
    color: #9ca3af;
  }

  .forecast-title {
    color: #f3f4f6;
    border-bottom-color: #374151;
  }

  .city-search {
    background-color: #111827;
    color: #f3f4f6;
    border-color: #374151;
  }

  .city-search::placeholder {
    color: #6b7280;
  }

  .city-search:focus {
    border-color: #60a5fa;
    box-shadow: 0 0 0 3px rgba(96, 165, 250, 0.15);
  }

  .search-results {
    background: #111827;
    border-color: #374151;
    box-shadow: 0 12px 30px rgba(0, 0, 0, 0.45);
  }

  .search-result {
    border-bottom-color: #1f2937;
  }

  .search-result:hover {
    background: #1f2937;
  }

  .search-result strong {
    color: #f3f4f6;
  }

  .search-result span {
    color: #9ca3af;
  }

  .location-button {
    background: #111827;
    color: #e5e7eb;
    border-color: #374151;
  }

  .location-button:hover {
    background: #1f2937;
    border-color: #60a5fa;
  }

  /* Menü & Dropdowns im Darkmode */
  .side-menu {
    background: #111827 !important;
  }

  .menu-separator {
    background-color: #374151;
  }

  .city-name {
    color: #f3f4f6;
  }

  .menu-city-row:hover {
    background: #1f2937;
  }

  .add-quick-btn {
    background: #1f2937;
    border-color: #374151;
    color: #9ca3af;
  }

  .add-quick-btn:hover {
    background: #3b82f6;
    color: white;
  }

  .dropdown-popup {
    background: #1f2937 !important;
    border-color: #374151 !important;
  }

  .dropdown-popup button {
    color: #f87171;
  }

  .dropdown-popup button:hover {
    background: #374151 !important;
  }

  .day-badge {
    background: #082f49;
    color: #38bdf8;
  }

  .weather-box,
  .daily-weather-box {
    scrollbar-color: #4b5563 #111827;
  }

  .weather-box::-webkit-scrollbar,
  .daily-weather-box::-webkit-scrollbar {
    height: 8px;
  }

  .weather-box::-webkit-scrollbar-track,
  .daily-weather-box::-webkit-scrollbar-track {
    background: #111827;
    border-radius: 10px;
  }

  .weather-box::-webkit-scrollbar-thumb,
  .daily-weather-box::-webkit-scrollbar-thumb {
    background: #4b5563;
    border-radius: 10px;
  }

  .weather-box::-webkit-scrollbar-thumb:hover,
  .daily-weather-box::-webkit-scrollbar-thumb:hover {
    background: #6b7280;
  }
}

/* =========================================================
   REDUZIERTE ANIMATIONEN
   ========================================================= */
@media (prefers-reduced-motion: reduce) {
  .hour-box,
  .city-search,
  .location-button,
  .search-result {
    transition: none;
  }

  .hour-box:hover {
    transform: none;
  }

  .weather-box,
  .daily-weather-box {
    scroll-behavior: auto;
  }
}
</style>