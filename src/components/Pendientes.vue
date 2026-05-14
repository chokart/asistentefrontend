<script setup lang="ts">
import { ref, onMounted, computed } from 'vue'
import axios from 'axios'

interface Pendiente {
  id?: number
  descripcion: string
  area: string
  responsable: string
  estado: string
  fechaCreacion?: string
  dias?: number
}

const API_URL = import.meta.env.VITE_API_URL || 'https://apitres.suiteminerals.com/api/pendientes'
const pendientes = ref<Pendiente[]>([])
const editandoId = ref<number | null>(null)
const filtroTexto = ref('')
const filtroEstado = ref('Todos')

const nuevoPendiente = ref<Pendiente>({
  descripcion: '',
  area: '',
  responsable: '',
  estado: 'Pendiente'
})

const pendientesFiltrados = computed(() => {
  return pendientes.value.filter(p => {
    const coincideTexto = 
      p.descripcion.toLowerCase().includes(filtroTexto.value.toLowerCase()) ||
      p.area.toLowerCase().includes(filtroTexto.value.toLowerCase()) ||
      p.responsable.toLowerCase().includes(filtroTexto.value.toLowerCase())
    
    const coincideEstado = filtroEstado.value === 'Todos' || p.estado === filtroEstado.value

    return coincideTexto && coincideEstado
  })
})

const cargarPendientes = async () => {
  try {
    const response = await axios.get(API_URL)
    pendientes.value = response.data
  } catch (error) {
    console.error('Error cargando pendientes:', error)
  }
}

const guardarPendiente = async () => {
  if (!nuevoPendiente.value.descripcion || !nuevoPendiente.value.responsable) {
    alert('Descripción y Responsable son obligatorios')
    return
  }
  try {
    if (editandoId.value) {
      await axios.put(`${API_URL}/${editandoId.value}`, nuevoPendiente.value)
    } else {
      await axios.post(API_URL, nuevoPendiente.value)
    }
    cancelarEdicion()
    cargarPendientes()
  } catch (error) {
    console.error('Error al guardar:', error)
  }
}

const editarPendiente = (p: Pendiente) => {
  editandoId.value = p.id!
  nuevoPendiente.value = { ...p }
}

const cancelarEdicion = () => {
  editandoId.value = null
  nuevoPendiente.value = { descripcion: '', area: '', responsable: '', estado: 'Pendiente' }
}

const completarPendiente = async (p: Pendiente) => {
  try {
    const copia = { ...p, estado: 'Completado' }
    await axios.put(`${API_URL}/${p.id}`, copia)
    cargarPendientes()
  } catch (error) {
    console.error('Error al completar:', error)
  }
}

const eliminarPendiente = async (id: number) => {
  if (!confirm('¿Eliminar este pendiente?')) return
  try {
    await axios.delete(`${API_URL}/${id}`)
    cargarPendientes()
  } catch (error) {
    console.error('Error al eliminar:', error)
  }
}

onMounted(cargarPendientes)
</script>

<template>
  <div class="container">
    <h1>Mis Pendientes del Trabajo</h1>

    <!-- Formulario -->
    <div class="card form-container">
      <h3>{{ editandoId ? 'Editar Pendiente' : 'Nuevo Pendiente' }}</h3>
      <div class="form-grid">
        <input v-model="nuevoPendiente.descripcion" placeholder="Descripción" type="text" />
        <input v-model="nuevoPendiente.area" placeholder="Área" type="text" />
        <input v-model="nuevoPendiente.responsable" placeholder="Responsable" type="text" />
        <select v-model="nuevoPendiente.estado">
          <option>Pendiente</option>
          <option>En Proceso</option>
          <option>Completado</option>
        </select>
        <div class="btn-group">
          <button @click="guardarPendiente" class="btn-primary">
            {{ editandoId ? 'Actualizar' : 'Agregar' }}
          </button>
          <button v-if="editandoId" @click="cancelarEdicion" class="btn-secondary">Cancelar</button>
        </div>
      </div>
    </div>

    <!-- Tabla -->
    <div class="card">
      <div class="filter-bar">
        <input v-model="filtroTexto" placeholder="🔍 Buscar por descripción, área o responsable..." type="text" class="search-input" />
        <select v-model="filtroEstado" class="status-filter">
          <option value="Todos">Todos los estados</option>
          <option>Pendiente</option>
          <option>En Proceso</option>
          <option>Completado</option>
        </select>
      </div>

      <div class="table-responsive">
        <table>
          <thead>
            <tr>
              <th>Descripción</th>
              <th>Área</th>
              <th>Responsable</th>
              <th>Estado</th>
              <th>Fecha Creación</th>
              <th>Días Activo</th>
              <th>Acciones</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="p in pendientesFiltrados" :key="p.id" :class="{ 'row-editing': editandoId === p.id }">
              <td>{{ p.descripcion }}</td>
              <td>{{ p.area }}</td>
              <td>{{ p.responsable }}</td>
              <td>
                <span :class="'badge ' + p.estado.toLowerCase().replace(' ', '-')">
                  {{ p.estado }}
                </span>
              </td>
              <td>{{ p.fechaCreacion }}</td>
              <td>{{ p.dias }}</td>
              <td>
                <div class="action-buttons">
                  <button v-if="p.estado !== 'Completado'" @click="completarPendiente(p)" class="btn-success" title="Completar">✓</button>
                  <button @click="editarPendiente(p)" class="btn-edit" title="Editar">✎</button>
                  <button @click="eliminarPendiente(p.id!)" class="btn-danger" title="Eliminar">🗑</button>
                </div>
              </td>
            </tr>
          </tbody>
        </table>
      </div>
      <p v-if="pendientesFiltrados.length === 0" style="text-align: center; padding: 1rem;">
        No se encontraron pendientes con esos criterios.
      </p>
    </div>
  </div>
</template>

<style scoped>
.container {
  max-width: 1200px;
  margin: 0 auto;
  padding: 2rem;
}

.card {
  background: white;
  border-radius: 8px;
  box-shadow: 0 2px 10px rgba(0,0,0,0.1);
  padding: 1.5rem;
  margin-bottom: 2rem;
  color: #333;
}

.form-grid {
  display: grid;
  grid-template-columns: 2fr 1fr 1fr 1fr auto;
  gap: 10px;
  align-items: center;
}

.filter-bar {
  display: flex;
  gap: 15px;
  margin-bottom: 20px;
}

.search-input {
  flex: 2;
  padding: 10px;
  font-size: 1rem;
}

.status-filter {
  flex: 1;
  padding: 10px;
}

.btn-group {
  display: flex;
  gap: 5px;
}

input, select {
  padding: 8px;
  border: 1px solid #ddd;
  border-radius: 4px;
  width: 100%; /* Asegura que ocupen el ancho de su celda */
  box-sizing: border-box;
}

/* Contenedor de tabla para scroll horizontal en móvil */
.table-responsive {
  overflow-x: auto;
  -webkit-overflow-scrolling: touch;
}

table {
  width: 100%;
  border-collapse: collapse;
  min-width: 600px; /* Evita que la tabla se comprima demasiado */
}

th, td {
  text-align: left;
  padding: 12px;
  border-bottom: 1px solid #eee;
}

th {
  background-color: #f8f9fa;
  font-weight: bold;
}

.row-editing {
  background-color: #fff9c4;
}

.action-buttons {
  display: flex;
  gap: 5px;
}

/* Botones y Badges */
.btn-primary { background: #41b883; color: white; border: none; padding: 8px 16px; border-radius: 4px; cursor: pointer; transition: background 0.3s; }
.btn-primary:hover { background: #3aa876; }
.btn-secondary { background: #9e9e9e; color: white; border: none; padding: 8px 16px; border-radius: 4px; cursor: pointer; }
.btn-success { background: #4caf50; color: white; border: none; padding: 6px 10px; border-radius: 4px; cursor: pointer; }
.btn-edit { background: #2196f3; color: white; border: none; padding: 6px 10px; border-radius: 4px; cursor: pointer; }
.btn-danger { background: #ff5252; color: white; border: none; padding: 6px 10px; border-radius: 4px; cursor: pointer; }

.badge {
  padding: 4px 8px;
  border-radius: 12px;
  font-size: 0.85rem;
  font-weight: bold;
  display: inline-block;
  white-space: nowrap;
}

.pendiente { background: #ffe082; color: #856404; }
.en-proceso { background: #bbdefb; color: #004085; }
.completado { background: #c8e6c9; color: #155724; }

/* === MEDIA QUERIES PARA RESPONSIVIDAD === */

@media (max-width: 992px) {
  .form-grid {
    grid-template-columns: 1fr 1fr; /* 2 columnas en tablets */
  }
  .form-grid > *:first-child {
    grid-column: span 2; /* Descripción ocupa todo el ancho */
  }
}

@media (max-width: 600px) {
  .container {
    padding: 1rem;
  }

  h1 {
    font-size: 1.5rem;
    text-align: center;
  }

  .form-grid {
    grid-template-columns: 1fr; /* 1 columna en celulares */
  }
  .form-grid > *:first-child {
    grid-column: span 1;
  }

  .btn-group {
    flex-direction: column;
  }

  .filter-bar {
    flex-direction: column; /* Filtros uno sobre otro */
    gap: 10px;
  }

  .search-input, .status-filter {
    flex: none;
    width: 100%;
  }

  .card {
    padding: 1rem;
  }
}
</style>
