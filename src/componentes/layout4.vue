<template>
  <div class="mapa-container">
    <h1 class="titulo">📍 Radar de Ocorrências Próximas</h1>

    <!-- PAINEL DE CONTROLE DO RAIO -->
    <div class="painel-busca">
      <div class="campo">
        <label>Raio de busca (metros):</label>
        <input v-model.number="raioBusca" type="number" min="50" step="500" />
      </div>
      <button @click="buscarOcorrenciasProximas" class="btn-atualizar" :disabled="carregandoGps">
        {{ carregandoGps ? '🔄 Buscando GPS...' : '🔍 Atualizar Mapa' }}
      </button>
    </div>

    <!-- ÁREA DO MAPA INTERATIVO -->
    <div class="mapa-wrapper">
      <div id="mapa-radar" class="mapa-elemento"></div>
    </div>
    
    <div class="status-barra">
      <span v-if="coordenadasCentro.lat">
        <b>Centro do Radar:</b> Lat {{ coordenadasCentro.lat.toFixed(4) }} | Lon {{ coordenadasCentro.lng.toFixed(4) }}
      </span>
      <span v-else class="aviso">Carregando posição de Brumado...</span>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount, nextTick } from 'vue';
import axios from 'axios';

// 🔥 IMPORTAÇÃO LOCAL COMPLETA: Importa o código e os estilos direto do node_modules instalados via npm
import L from 'leaflet';
import 'leaflet/dist/leaflet.css';

const API_URL = 'https://tide-api-c6t2.onrender.com/tide-api/ocorrencias/proximas';

const raioBusca = ref(5000); 
const carregandoGps = ref(false);
const ocorrenciasProximas = ref([]);

const coordenadasCentro = ref({
  lat: -14.2045, 
  lng: -41.6644
});

let mapaInstancia = null;
let camadaElementos = null;
let observadorRedimensionamento = null;

// Corrige o caminho visual das imagens dos pins usando URLs de fallback confiáveis do OpenStreetMap
const configurarIconesMarcadores = () => {
  delete L.Icon.Default.prototype._getIconUrl;
  L.Icon.Default.mergeOptions({
    iconRetinaUrl: 'https://cloudflare.com',
    iconUrl: 'https://cloudflare.com',
    shadowUrl: 'https://cloudflare.com',
  });
};

const inicializarMapa = () => {
  configurarIconesMarcadores();

  // Cria a instância do mapa associando à div#mapa-radar
  mapaInstancia = L.map('mapa-radar', {
    trackResize: true
  }).setView([coordenadasCentro.value.lat, coordenadasCentro.value.lng], 14);

  L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
    attribution: '© OpenStreetMap'
  }).addTo(mapaInstancia);

  camadaElementos = L.layerGroup().addTo(mapaInstancia);

  // Monitora o contêiner para redesenhar o mapa assim que ganhar dimensões físicas na tela
  const mapaEl = document.getElementById('mapa-radar');
  if (mapaEl && 'ResizeObserver' in window) {
    observadorRedimensionamento = new ResizeObserver(() => {
      if (mapaInstancia) {
        mapaInstancia.invalidateSize();
      }
    });
    observadorRedimensionamento.observe(mapaEl);
  }

  setTimeout(() => {
    if (mapaInstancia) mapaInstancia.invalidateSize();
  }, 150);
};

const desenharMapa = () => {
  if (!camadaElementos) return;

  camadaElementos.clearLayers();

  // 1. Círculo de alcance do radar
  L.circle([coordenadasCentro.value.lat, coordenadasCentro.value.lng], {
    color: '#dc2626',
    fillColor: '#ef4444',
    fillOpacity: 0.1,
    radius: raioBusca.value
  }).addTo(camadaElementos);

  // 2. Marcador central do monitoramento
  L.marker([coordenadasCentro.value.lat, coordenadasCentro.value.lng])
    .addTo(camadaElementos)
    .bindPopup('<b>📍 Ponto Central de Monitoramento</b>')
    .openPopup();

  // 3. Marcadores de crimes recuperados da sua API Spring Boot
  ocorrenciasProximas.value.forEach(oc => {
    if (oc.localizacao && oc.localizacao.latitude) {
      const popupHtml = `
        <div style="font-family: sans-serif; font-size: 13px; line-height: 1.4;">
          <h4 style="margin: 0 0 5px 0; color: #dc2626;">⚠️ Ocorrência</h4>
          <b>Vítima:</b> ${oc.nomeVitima}<br/>
          <b>Relato:</b> ${oc.relato}<br/>
          <small style="color: #777;">Data: ${new Date(oc.dataHora).toLocaleString('pt-BR')}</small>
        </div>
      `;

      L.marker([oc.localizacao.latitude, oc.localizacao.longitude])
        .addTo(camadaElementos)
        .bindPopup(popupHtml);
    }
  });
};

const obterGpsAtual = () => {
  if (!navigator.geolocation) {
    buscarOcorrenciasProximas();
    return;
  }

  carregandoGps.value = true;
  navigator.geolocation.getCurrentPosition(
    (position) => {
      coordenadasCentro.value.lat = position.coords.latitude;
      coordenadasCentro.value.lng = position.coords.longitude;
      carregandoGps.value = false;
      
      if (mapaInstancia) {
        mapaInstancia.setView([position.coords.latitude, position.coords.longitude], 14);
      }
      buscarOcorrenciasProximas();
    },
    (error) => {
      console.warn('Usando coordenadas fixas de Brumado - BA devido ao GPS indisponível.', error);
      carregandoGps.value = false;
      buscarOcorrenciasProximas();
    },
    { timeout: 4000 }
  );
};

const buscarOcorrenciasProximas = async () => {
  try {
    const response = await axios.get(API_URL, {
      params: {
        latitude: coordenadasCentro.value.lat,
        longitude: coordenadasCentro.value.lng,
        raio: raioBusca.value
      }
    });
    ocorrenciasProximas.value = response.data;
    desenharMapa();
  } catch (error) {
    console.error('Erro de conexão com o back-end:', error);
  }
};

onMounted(async () => {
  await nextTick();
  inicializarMapa();
  obterGpsAtual();
});

onBeforeUnmount(() => {
  if (observadorRedimensionamento) {
    observadorRedimensionamento.disconnect();
  }
});
</script>

<style scoped>
.mapa-container {
  max-width: 800px;
  margin: 20px auto;
  font-family: system-ui, -apple-system, sans-serif;
  padding: 0 15px;
}
.titulo {
  font-size: 22px;
  color: #1e293b;
  text-align: center;
  margin-bottom: 20px;
}
.painel-busca {
  display: flex;
  background: #ffffff;
  padding: 15px;
  border-radius: 8px;
  box-shadow: 0 4px 10px rgba(0,0,0,0.05);
  gap: 15px;
  align-items: flex-end;
  margin-bottom: 15px;
  border: 1px solid #e2e8f0;
}
.campo {
  display: flex;
  flex-direction: column;
  flex: 1;
}
.campo label {
  font-size: 13px;
  font-weight: 600;
  color: #475569;
  margin-bottom: 6px;
}
.campo input {
  padding: 10px;
  border: 1px solid #cbd5e1;
  border-radius: 6px;
  font-size: 14px;
}
.btn-atualizar {
  background: #4f46e5;
  color: white;
  border: none;
  padding: 11px 20px;
  border-radius: 6px;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s;
}
.btn-atualizar:hover { background: #4338ca; }
.btn-atualizar:disabled { background: #cbd5e1; cursor: not-allowed; }

.mapa-wrapper {
  width: 100%;
  clear: both;
  display: block;
}
.mapa-elemento {
  display: block;
  height: 450px;
  width: 100%;
  background-color: #cbd5e1;
  border-radius: 8px;
  box-shadow: 0 4px 12px rgba(0,0,0,0.1);
}

/* Evita distorção visual nos blocos de imagem do OpenStreetMap herdada por frameworks CSS */
.mapa-elemento :deep(img) {
  max-height: none !important;
  max-width: none !important;
}
.status-barra {
  margin-top: 10px;
  font-size: 12px;
  color: #64748b;
  text-align: right;
}
.aviso { color: #f59e0b; }
</style>
