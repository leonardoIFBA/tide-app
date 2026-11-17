<template>
  <div class="container">
    <h1>📍 Sistema TIDE - Registro de Ocorrências</h1>
    <div>
      <MapaOcorrencias />
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
import { ref, onMounted } from 'vue';
import axios from 'axios';
import MapaOcorrencias from './MapaOcorrencias.vue';

// URL base da sua API Java (Spring Boot)
const API_URL = 'https://tide-api-c6t2.onrender.com/tide-api/ocorrencias';

const carregandoGps = ref(false);
const raioBusca = ref(1000); // 1000 metros (1km) padrão
const ocorrenciasProximas = ref([]);

const novaOcorrencia = ref({
  nomeVitima: '',
  relato: '',
  dataHora: '',
  latitude: null,
  longitude: null
});

// 💡 Nome corrigido e garantido para bater com o HTML: obterLocalizacao
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
      buscarOcorrenciasProximas();
    },
    (error) => {
      console.error('Erro de geolocalização:', error);
      carregandoGps.value = false;
      
      // Modo de segurança para teste no Notebook (Injeta coordenadas fictícias se o GPS falhar)
      novaOcorrencia.value.latitude = -14.2049;  // Coordenadas aproximadas de teste (Brumado/BA)
      novaOcorrencia.value.longitude = -41.6652;
      buscarOcorrenciasProximas();
    },
    { enableHighAccuracy: true, timeout: 5000 }
  );
};

// Envia os dados estruturados no formato do seu OcorrenciaDTO
const cadastrarOcorrencia = async () => {
  try {
    // Insere o timestamp atual formatado no padrão LocalDateTime do Java
    novaOcorrencia.value.dataHora = new Date().toISOString().replace(/\.\d{3}Z$/, ""); 

    await axios.post(API_URL, novaOcorrencia.value);
    alert('Ocorrência registrada com sucesso!');
    
    // Limpa o formulário mantendo o GPS para novas buscas
    novaOcorrencia.value.nomeVitima = '';
    novaOcorrencia.value.relato = '';
    
    buscarOcorrenciasProximas();
  } catch (error) {
    console.error('Erro ao enviar:', error);
    alert('Erro ao conectar com o back-end Java. Verifique se o CORS está liberado.');
  }
};

// Chama o endpoint de proximidade via parâmetros de URL (@RequestParam)
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

// Executa automaticamente assim que a página termina de carregar
onMounted(() => {
  obterLocalizacao();
});
</script>


<style scoped>
.container {
  max-width: 600px;
  margin: 30px auto;
  font-family: Arial, sans-serif;
  padding: 0 15px;
}
h1 { text-align: center; color: #2c3e50; font-size: 24px; }
h2 { color: #34495e; border-bottom: 2px solid #ecf0f1; padding-bottom: 8px; font-size: 18px; }
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
</style>
