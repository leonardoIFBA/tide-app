<template>
  <div class="container">
    <h1>📍 Sistema TIDE - Registro de Ocorrências</h1>

    <!-- NOVO: CONTAINER DO MAPA INTERATIVO -->
    <div class="card mapa-card">
      <h2>Mapa de Ocorrências (Brumado - BA)</h2>
      <div id="map" class="meu-mapa"></div>
    </div>

    <!-- SEÇÃO 1: REGISTRAR NOVA OCORRÊNCIA -->
    <div class="card">
      <h2>Nova Ocorrência</h2>
      <form @submit.prevent="cadastrarOcorrencia">
        <div class="form-group">
          <label>Nome da Vítima:</label>
          <input v-model="novaOcorrencia.nomeVitima" type="text" required placeholder="Digite o nome..." />
        </div>

        <div class="form-group">
          <label>Relato do Incidente:</label>
          <textarea v-model="novaOcorrencia.relato" required placeholder="O que aconteceu?..."></textarea>
        </div>

        <div class="geo-box">
          <p><strong>Sua Localização Atual:</strong></p>
          <span v-if="carregandoGps">🔄 Obtendo coordenadas do GPS...</span>
          <span v-else-if="novaOcorrencia.latitude && novaOcorrencia.longitude" class="coordenadas-texto">
            Lat: {{ novaOcorrencia.latitude.toFixed(5) }} | Lon: {{ novaOcorrencia.longitude.toFixed(5) }}
          </span>
          <span v-else class="erro">⚠️ GPS não capturado. Clique no botão abaixo.</span>
          
          <button type="button" @click="obterLocalizacao" class="btn-geo">Capturar GPS Atual</button>
        </div>

        <button type="submit" class="btn-enviar" :disabled="!novaOcorrencia.latitude">Enviar Ocorrência</button>
      </form>
    </div>

    <!-- SEÇÃO 2: BUSCAR OCORRÊNCIAS PRÓXIMAS -->
    <div class="card">
      <h2>Ocorrências no Raio de Proximidade</h2>
      <div class="filtro-raio">
        <label>Raio de busca (metros):</label>
        <input v-model.number="raioBusca" type="number" min="10" />
        <button @click="buscarOcorrenciasProximas" class="btn-buscar" :disabled="!novaOcorrencia.latitude">
          Buscar Próximas
        </button>
      </div>

      <p v-if="ocorrenciasProximas.length === 0" class="aviso">Nenhuma ocorrência encontrada neste raio ou GPS não capturado.</p>
      
      <ul class="lista-ocorrencias">
        <li v-for="oc in ocorrenciasProximas" :key="oc.id" class="item-ocorrencia">
          <strong>Vítima:</strong> {{ oc.nomeVitima }} <br />
          <strong>Relato:</strong> {{ oc.relato }} <br />
          <small>📅 {{ formatarData(oc.dataHora) }} | 🗺️ Lon: {{ oc.localizacao.longitude }} Lat: {{ oc.localizacao.latitude }}</small>
        </li>
      </ul>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, watch } from 'vue';
import axios from 'axios';

// URL base da sua API Java (Spring Boot)
const API_URL = 'http://localhost:8080/tide-api/ocorrencias';

const carregandoGps = ref(false);
const raioBusca = ref(5000); // Alterado para 5km padrão para ver todos os dados fictícios de Brumado
const ocorrenciasProximas = ref([]);

const novaOcorrencia = ref({
  nomeVitima: '',
  relato: '',
  dataHora: '',
  latitude: null,
  longitude: null
});

// Referências para controle do Leaflet
let mapaInstancia = null;
let camadaMarcadores = null;

// Função para iniciar e configurar o mapa Leaflet
const inicializarMapa = () => {
  if (typeof window.L === 'undefined') {
    console.error('Leaflet não foi carregado. Certifique-se de adicioná-lo ao index.html');
    return;
  }

  corrigirIconesLeaflet();

  // Inicializa o mapa normalmente
  mapaInstancia = window.L.map('map').setView([-14.2045, -41.6644], 14);

  window.L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
    attribution: '© OpenStreetMap contributors'
  }).addTo(mapaInstancia);

  camadaMarcadores = window.L.layerGroup().addTo(mapaInstancia);

  // 🔥 SOLUÇÃO PARA O ERRO CINZA:
  // Aguarda 200ms para garantir que o layout CSS do Vue terminou de se assentar na tela
  // e força o Leaflet a recalcular o espaço disponível para renderizar os blocos.
  setTimeout(() => {
    if (mapaInstancia) {
      mapaInstancia.invalidateSize();
    }
  }, 200);
};

// Atualiza os marcadores exibidos no mapa baseado na lista de ocorrências próximas
const atualizarMarcadoresNoMapa = () => {
  if (!camadaMarcadores || typeof window.L === 'undefined') return;

  // Remove os marcadores antigos para não duplicar na tela
  camadaMarcadores.clearLayers();

  // Se o usuário capturou o próprio GPS, desenha um círculo vermelho simulando o raio de busca
  if (novaOcorrencia.value.latitude && novaOcorrencia.value.longitude) {
    window.L.circle([novaOcorrencia.value.latitude, novaOcorrencia.value.longitude], {
      color: '#ef4444',
      fillColor: '#fca5a5',
      fillOpacity: 0.15,
      radius: raioBusca.value
    }).addTo(camadaMarcadores);

    // Adiciona uma estrela ou marcador diferenciado no ponto exato do usuário
    window.L.marker([novaOcorrencia.value.latitude, novaOcorrencia.value.longitude])
      .addTo(camadaMarcadores)
      .bindPopup('<b>Você está aqui 🌟</b>')
      .openPopup();
  }

  // Percorre as ocorrências vindas do Spring Boot e adiciona um balão informativo no mapa
  ocorrenciasProximas.value.forEach(oc => {
    if (oc.localizacao && oc.localizacao.latitude) {
      const textoPopup = `
        <div style="font-family: Arial, sans-serif; font-size: 13px;">
          <strong style="color: #ef4444;">⚠️ Ocorrência registrada</strong><br/>
          <b>Vítima:</b> ${oc.nomeVitima}<br/>
          <b>Relato:</b> ${oc.relato}<br/>
          <small style="color: #666;">Data: ${formatarData(oc.dataHora)}</small>
        </div>
      `;

      window.L.marker([oc.localizacao.latitude, oc.localizacao.longitude])
        .addTo(camadaMarcadores)
        .bindPopup(textoPopup);
    }
  });
};

// Monitora mudanças na lista para redesenhar os pontos automaticamente no mapa
watch(ocorrenciasProximas, () => {
  atualizarMarcadoresNoMapa();
}, { deep: true });

const obterLocalizacao = () => {
  if (!navigator.geolocation) {
    alert('Seu navegador não suporta Geolocalização nativa.');
    return;
  }

  carregandoGps.value = true;

  navigator.geolocation.getCurrentPosition(
    (position) => {
      novaOcorrencia.value.latitude = position.coords.latitude;
      novaOcorrencia.value.longitude = position.coords.longitude;
      carregandoGps.value = false;
      
      // Move o foco do mapa para onde o usuário se localiza
      if (mapaInstancia) {
        mapaInstancia.setView([position.coords.latitude, position.coords.longitude], 14);
      }
      buscarOcorrenciasProximas();
    },
    (error) => {
      console.error('Erro de geolocalização:', error);
      carregandoGps.value = false;
      
      // Modo de segurança para teste local em Brumado/BA se o GPS falhar
      novaOcorrencia.value.latitude = -14.2045;  
      novaOcorrencia.value.longitude = -41.6644;
      buscarOcorrenciasProximas();
    },
    { enableHighAccuracy: true, timeout: 5000 }
  );
};

const cadastrarOcorrencia = async () => {
  try {
    novaOcorrencia.value.dataHora = new Date().toISOString().replace(/\.\d{3}Z$/, ""); 

    await axios.post(API_URL, novaOcorrencia.value);
    alert('Ocorrência registrada com sucesso!');
    
    novaOcorrencia.value.nomeVitima = '';
    novaOcorrencia.value.relato = '';
    
    buscarOcorrenciasProximas();
  } catch (error) {
    console.error('Erro ao enviar:', error);
    alert('Erro ao conectar com o back-end Java. Verifique se o CORS está liberado.');
  }
};

const buscarOcorrenciasProximas = async () => {
  if (!novaOcorrencia.value.latitude || !novaOcorrencia.value.longitude) return;

  try {
    const response = await axios.get(`${API_URL}/proximas`, {
      params: {
        latitude: novaOcorrencia.value.latitude,
        longitude: novaOcorrencia.value.longitude,
        raio: raioBusca.value
      }
    });
    ocorrenciasProximas.value = response.data;
  } catch (error) {
    console.error('Erro ao buscar próximas:', error);
  }
};

const formatarData = (dataStr) => {
  if (!dataStr) return '';
  return new Date(dataStr).toLocaleString('pt-BR');
};

onMounted(() => {
  inicializarMapa();
  obterLocalizacao();
});
</script>


<style scoped>
/* ESTILOS NOVOS DO MAPA */
.mapa-card {
  padding: 10px !important;
}
.meu-mapa {
  height: 350px;
  width: 100%;
  border-radius: 6px;
  background-color: #e5e7eb;
  z-index: 1; /* Garante que o mapa fique abaixo de menus dropdown se houver */
}

/* ESTILOS QUE VOCÊ JÁ TINHA */
.container {
  max-width: 600px;
  margin: 30px auto;
  font-family: Arial, sans-serif;
  padding: 0 15px;
}
h1 { text-align: center; color: #2c3e50; font-size: 24px; }
h2 { color: #34495e; border-bottom: 2px solid #ecf0f1; padding-bottom: 8px; font-size: 18px; margin-top: 5px;}
.card {
  background: #fff;
  border-radius: 8px;
  padding: 20px;
  margin-bottom: 20px;
  box-shadow: 0 4px 6px rgba(0,0,0,0.1);
}
.form-group { margin-bottom: 15px; display: flex; flex-direction: column; }
label { font-weight: bold; margin-bottom: 5px; color: #555; }
input[type="text"], input[type="number"], textarea {
  padding: 10px; border: 1px solid #ccc; border-radius: 4px; font-size: 14px;
}
textarea { height: 80px; resize: vertical; }
.geo-box {
  background: #f8f9fa; border: 1px dashed #cbd5e1; padding: 12px; border-radius: 6px; margin-bottom: 15px; text-align: center;
}
.coordenadas-texto { color: #16a34a; font-weight: bold; display: block; margin-bottom: 8px; }
.btn-geo { background: #3b82f6; color: white; border: none; padding: 6px 12px; border-radius: 4px; cursor: pointer; }
.btn-enviar { background: #10b981; color: white; width: 100%; border: none; padding: 12px; border-radius: 4px; font-size: 16px; font-weight: bold; cursor: pointer; }
.btn-enviar:disabled, .btn-buscar:disabled { background: #cbd5e1; cursor: not-allowed; }
.filtro-raio { display: flex; gap: 10px; align-items: center; margin-bottom: 15px; }
.filtro-raio input { width: 100px; }
.btn-buscar { background: #6366f1; color: white; border: none; padding: 10px 15px; border-radius: 4px; cursor: pointer; }
.lista-ocorrencias { list-style: none; padding: 0; }
.item-ocorrencia { background: #f1f5f9; padding: 12px; border-radius: 6px; margin-bottom: 10px; border-left: 4px solid #6366f1; }
.erro { color: #ef4444; display: block; margin-bottom: 8px; font-size: 14px; }
.aviso { color: #64748b; font-style: italic; text-align: center; }

/* Garante que nenhum estilo externo quebre a renderização dos blocos do mapa */
.meu-mapa :deep(img) {
  max-height: none !important;
  max-width: none !important;
}
</style>
