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
    <div id="mapa-radar" class="mapa-elemento"></div>
    
    <div class="status-barra">
      <span v-if="coordenadasCentro.lat">
        <b>Centro do Radar:</b> Lat {{ coordenadasCentro.lat.toFixed(4) }} | Lon {{ coordenadasCentro.lng.toFixed(4) }}
      </span>
      <span v-else class="aviso">Carregando posição de Brumado...</span>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, nextTick } from 'vue';
import axios from 'axios';

const API_URL = 'http://localhost:8080/tide-api/ocorrencias/proximas';

const raioBusca = ref(5000); // Raio padrão de 5km
const carregandoGps = ref(false);
const ocorrenciasProximas = ref([]);

const coordenadasCentro = ref({
  lat: -14.2045, // Brumado - Centro
  lng: -41.6644
});

let mapaInstancia = null;
let camadaElementos = null;

// Função que injeta os scripts do Leaflet diretamente na execução do componente
const carregarLeafletDinamicamente = () => {
  return new Promise((resolve) => {
    if (window.L) {
      resolve();
      return;
    }

    // Injeta o arquivo CSS do Leaflet
    const link = document.createElement('link');
    link.rel = 'stylesheet';
    link.href = 'https://unpkg.com';
    document.head.appendChild(link);

    // Injeta o arquivo Javascript do Leaflet
    const script = document.createElement('script');
    script.src = 'https://unpkg.com';
    script.onload = () => resolve();
    document.head.appendChild(script);
  });
};

const configurarIconesMarcadores = () => {
  if (!window.L) return;
  delete window.L.Icon.Default.prototype._getIconUrl;
  window.L.Icon.Default.mergeOptions({
    iconRetinaUrl: 'https://unpkg.com',
    iconUrl: 'https://unpkg.com',
    shadowUrl: 'https://unpkg.com',
  });
};

const inicializarMapa = () => {
  configurarIconesMarcadores();

  // Cria a instância apontando para a div#mapa-radar
  mapaInstancia = window.L.map('mapa-radar').setView([coordenadasCentro.value.lat, coordenadasCentro.value.lng], 14);

  window.L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
    attribution: '© OpenStreetMap'
  }).addTo(mapaInstancia);

  camadaElementos = window.L.layerGroup().addTo(mapaInstancia);

  // Corrige o bug do contêiner cinza forçando o Leaflet a ler o tamanho correto da tela
  setTimeout(() => {
    if (mapaInstancia) mapaInstancia.invalidateSize();
  }, 300);
};

const desenharMapa = () => {
  if (!camadaElementos || !window.L) return;

  // Limpa tudo o que já estava desenhado
  camadaElementos.clearLayers();

  // 1. Desenha o círculo de alcance (vermelho) baseado no raio digitado
  window.L.circle([coordenadasCentro.value.lat, coordenadasCentro.value.lng], {
    color: '#dc2626',
    fillColor: '#ef4444',
    fillOpacity: 0.1,
    radius: raioBusca.value
  }).addTo(camadaElementos);

  // 2. Coloca uma estrela ou marcador na posição atual da busca
  window.L.marker([coordenadasCentro.value.lat, coordenadasCentro.value.lng])
    .addTo(camadaElementos)
    .bindPopup('<b>📍 Ponto Central de Monitoramento</b>')
    .openPopup();

  // 3. Adiciona as ocorrências vindas do banco de dados (Supabase/Spring)
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

      window.L.marker([oc.localizacao.latitude, oc.localizacao.longitude])
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
      console.warn('GPS não acessível, mantendo o radar fixo em Brumado-BA.', error);
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
    console.error('Falha ao conectar com o back-end:', error);
  }
};

onMounted(async () => {
  await nextTick();
  // Aguarda baixar os arquivos do Leaflet antes de tentar renderizar a tela
  await carregarLeafletDinamicamente();
  inicializarMapa();
  obterGpsAtual();
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

.mapa-elemento {
  height: 450px;
  width: 100%;
  border-radius: 8px;
  box-shadow: 0 4px 12px rgba(0,0,0,0.1);
  background-color: #cbd5e1;
}

/* Força os blocos gráficos a renderizarem sem quebras */
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
