<template>
  <div id="map-container">
    <div id="map"></div>

    <div class="dev-controls">
      <button class="dev-btn" @click="toggleDevMode">
        {{ isDevMode ? 'Modo Dev: ON' : 'Modo Dev: OFF' }}
      </button>
      <button v-if="isDevMode" class="dev-btn" @click="resetApplication">
        Resetar (R)
      </button>
    </div>

    <button class="iniciar-btn" v-if="!monitorando" @click="abrirSelecaoVeiculo">
      Iniciar monitoramento
    </button>

    <div v-if="mostrarSelecao" class="modal-overlay">
      <div class="modal-content">
        <h3>Selecione o tipo de veículo</h3>
        <ul class="lista-veiculos">
          <li v-for="(label, key) in tiposVeiculos" :key="key">
            <button class="veiculo-btn" @click="selecionarVeiculo(key)">
              {{ label }}
            </button>
          </li>
        </ul>
      </div>
    </div>
  </div>
</template>

<script>
import L from "leaflet"
import "leaflet/dist/leaflet.css"
import "@/assets/alertasBauru.css" 

export default {
  name: "AlertasBauru",
  data() {
    return {
      map: null,
      userMarker: null,
      arrowElement: null,
      risco: null,
      apiUrl: "http://localhost:8000/calcular_risco",
      monitorando: false,
      mostrarSelecao: false,
      tipoSelecionado: null,
      intervalId: null,
      tiposVeiculos: {
        tp_veiculo_automovel: "Automóvel",
        tp_veiculo_motocicleta: "Motocicleta",
        tp_veiculo_bicicleta: "Bicicleta",
        tp_veiculo_caminhao: "Caminhão",
        tp_veiculo_onibus: "Ônibus",
        tp_veiculo_outros: "Outros"
      },
      audioContext: null,
      lastAlertAt: 0,
      alertCooldownMs: 10000,
      previousInterpretacao: null,
      isDevMode: false, // Flag do DevMode 
      simulatedCoords: {
        latitude: -22.3310,
        longitude: -49.0579,
        heading: 0
      },
      clickMarker: null,
    }
  },
  mounted() {
    this.initMap();
    window.addEventListener('keydown', this.handleKeyDown);
  },

  beforeUnmount() {
    window.removeEventListener('keydown', this.handleKeyDown);
},



  methods: {

    handleKeyDown(event) {
        if (event.key === 'r' || event.key === 'R') {
            if (this.isDevMode) { 
                this.resetApplication();
            }
        }
    },

    initMap() {
      this.map = L.map("map").setView([-22.3145, -49.0587], 13)
      L.tileLayer("https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png", {
        attribution: "© OpenStreetMap"
      }).addTo(this.map)

      if (this.isDevMode) {
        this.map.on('click', this.onMapClick);
        console.warn("MODO DEV ATIVO. Clique no mapa para simular.");
      }
    },
    
    // ------------------------------------------------------------------
    // NOVOS MÉTODOS DE CONTROLE
    // ------------------------------------------------------------------

    toggleDevMode() {
        this.isDevMode = !this.isDevMode;

        if (this.isDevMode) {
            this.map.on('click', this.onMapClick);
            console.warn("MODO DEV ATIVADO. Clique no mapa para simular.");
        } else {
            this.map.off('click', this.onMapClick);
            console.warn("MODO DEV DESATIVADO.");
            
            if (this.clickMarker) {
                this.map.removeLayer(this.clickMarker);
                this.clickMarker = null;
            }
        }
        this.resetApplication();
    },

    resetApplication() {
        if (this.intervalId) {
            clearInterval(this.intervalId);
            this.intervalId = null;
        }

        if (this.userMarker) {
            this.map.removeLayer(this.userMarker);
            this.userMarker = null;
            this.arrowElement = null;
        }
        if (this.clickMarker) {
            this.map.removeLayer(this.clickMarker);
            this.clickMarker = null;
        }
        
        this.monitorando = false;
        this.risco = null;
        this.tipoSelecionado = null;
        this.previousInterpretacao = null;
        
        const defaultLat = -22.3310;
        const defaultLng = -49.0579;
        this.map.setView([defaultLat, defaultLng], 13);
        
        this.simulatedCoords = {
            latitude: defaultLat,
            longitude: defaultLng,
            heading: 0
        };

        console.log("Aplicação resetada com sucesso.");
    },

    // ------------------------------------------------------------------
    // Métodos Existentes
    // ------------------------------------------------------------------

    abrirSelecaoVeiculo() {
      this.mostrarSelecao = true
    },

    selecionarVeiculo(tipo) {
      this.tipoSelecionado = tipo
      this.mostrarSelecao = false
      this.monitorando = true
      try {
        const AudioContext = window.AudioContext || window.webkitAudioContext
        if (!this.audioContext && AudioContext) this.audioContext = new AudioContext()
      } catch (e) {
        console.error("Erro ao inicializar áudio:", e)
      }
      this.iniciarGeolocalizacao()
    },

    onMapClick(e) {
      const lat = e.latlng.lat;
      const lng = e.latlng.lng;

      if (this.clickMarker) {
        this.map.removeLayer(this.clickMarker);
      }

      const icon = L.divIcon({
        className: "click-location-icon",
        html: '📍',
        iconSize: [40, 40], 
        iconAnchor: [20, 40] 
      });

      this.clickMarker = L.marker([lat, lng], { icon: icon }).addTo(this.map);

      const container = L.DomUtil.create('div');
      const title = L.DomUtil.create('b', '', container);
      title.innerText = 'Simular coordenadas:';
      const coords = L.DomUtil.create('p', '', container);
      coords.innerText = `Lat: ${lat.toFixed(4)}, Lng: ${lng.toFixed(4)}`;
      const button = L.DomUtil.create('button', '', container);
      button.innerText = 'Mover usuário';
      button.style = "background:#007bff; color:white; border:none; padding: 5px 10px; border-radius: 5px; cursor:pointer;";

      const vm = this;
      L.DomEvent.on(button, 'click', (ev) => {
          L.DomEvent.stopPropagation(ev);
          vm.moverUsuarioDev(lat, lng);
      });

      this.clickMarker.bindPopup(container).openPopup();
    },

    moverUsuarioDev(lat, lng) {
      this.simulatedCoords.latitude = lat;
      this.simulatedCoords.longitude = lng;

      if (this.clickMarker) {
        this.map.closePopup(this.clickMarker.getPopup());
        this.map.removeLayer(this.clickMarker);
        this.clickMarker = null;
      }
      
      this.atualizarPosicao(lat, lng, this.simulatedCoords.heading);
      this.map.setView([lat, lng], 18);

      if (this.monitorando) {
        this.consultarRisco(this.simulatedCoords.latitude, this.simulatedCoords.longitude);
      }
    },

    iniciarGeolocalizacao() {
      // --- Bloco do Modo Dev ---
      if (this.isDevMode) {
        this.atualizarPosicao(this.simulatedCoords.latitude, this.simulatedCoords.longitude, this.simulatedCoords.heading);

        if (!this.intervalId) {
            this.consultarRisco(this.simulatedCoords.latitude, this.simulatedCoords.longitude);
            this.intervalId = setInterval(() => {
                this.consultarRisco(this.simulatedCoords.latitude, this.simulatedCoords.longitude);
            }, 30000);
        }
        return; 
      }

      // --- Lógica de Produção (Geolocalização Real) ---
      if (!navigator.geolocation) {
        alert("Geolocalização não suportada pelo navegador.")
        return
      }

      navigator.geolocation.watchPosition(
        pos => {
          const lat = pos.coords.latitude
          const lng = pos.coords.longitude
          const heading = pos.coords.heading
          this.atualizarPosicao(lat, lng, heading)

          if (!this.intervalId) {
            this.consultarRisco(lat, lng)
            this.intervalId = setInterval(() => {
              this.consultarRisco(lat, lng) 
            }, 30000)
          }
        },
        err => console.error("Erro ao obter localização:", err),
        { enableHighAccuracy: true, maximumAge: 0 }
      )
    },

    atualizarPosicao(lat, lng, heading) {
      if (!this.userMarker) {
        const arrowIcon = L.divIcon({
          className: "user-arrow-icon",
          html: '<div class="arrow"></div>',
          iconSize: [30, 30],
          iconAnchor: [15, 15]
        })
        this.userMarker = L.marker([lat, lng], { icon: arrowIcon }).addTo(this.map)
        this.arrowElement = this.userMarker.getElement().querySelector(".arrow")
        this.map.setView([lat, lng], 18)
      } else {
        this.userMarker.setLatLng([lat, lng])
      }
      if (heading !== null && !isNaN(heading) && this.arrowElement) {
        this.arrowElement.style.transform = `rotate(${heading}deg)`
      }
    },

    async consultarRisco(lat, lng) {
      if (!this.tipoSelecionado) {
          console.warn("Tipo de veículo não selecionado. Consulta abortada.");
          return;
      }
      try {
        const payload = {
          latitude: lat,
          longitude: lng,
          tp_veiculo_selecionado: this.tipoSelecionado
        }
        const res = await fetch(this.apiUrl, {
          method: "POST",
          headers: { "Content-Type": "application/json" },
          body: JSON.stringify(payload)
        })
        if (!res.ok) throw new Error(`Erro ${res.status}: ${res.statusText}`)
        const data = await res.json()
        this.risco = data.risco_estimado
        this.exibirRiscoNoMapa(this.risco, data.interpretacao)
      } catch (error) {
        console.error("Erro ao consultar risco:", error)
      }
    },

    playAlertSound(duration = 0.5, frequency = 880) {
      // ... (código playAlertSound) ...
      try {
        if (!this.audioContext) {
          const AudioContext = window.AudioContext || window.webkitAudioContext
          this.audioContext = new AudioContext()
        }
        const now = this.audioContext.currentTime
        const osc = this.audioContext.createOscillator()
        const gain = this.audioContext.createGain()
        osc.type = "sine"
        osc.frequency.setValueAtTime(frequency, now)
        gain.gain.setValueAtTime(0, now)
        gain.gain.linearRampToValueAtTime(0.9, now + 0.01)
        gain.gain.exponentialRampToValueAtTime(0.001, now + duration)
        osc.connect(gain)
        gain.connect(this.audioContext.destination)
        osc.start(now)
        osc.stop(now + duration)
      } catch (e) {
        console.error("Erro ao tentar reproduzir som de alerta:", e)
      }
    },

    exibirRiscoNoMapa(risco, interpretacao) {
      if (!this.arrowElement) return
      let cor = "#007bff"
      if (interpretacao === "MÉDIO") cor = "#ffcc00"
      if (interpretacao === "ALTO") cor = "#ff0000"
      this.arrowElement.style.borderBottomColor = cor
      if (this.userMarker) {
        this.userMarker
          .bindPopup(`Risco estimado: <b>${interpretacao}</b> (${(risco * 100).toFixed(1)}%)`)
          .openPopup()
      }
      const now = Date.now()
      const podeAlertar =
        interpretacao === "ALTO" &&
        (this.previousInterpretacao !== "ALTO" || now - this.lastAlertAt > this.alertCooldownMs)
      if (podeAlertar) {
        this.playAlertSound(0.6, 880)
        this.lastAlertAt = now
      }
      this.previousInterpretacao = interpretacao
    }
  }
}
</script>
