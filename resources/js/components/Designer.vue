<template>
    <header-component
        :addTable="addTable"
        :openImportModal="openImportModal"
        :openExportModal="openExportModal"
        :openDiagramsModal="openDiagramsModal"
        :saveDiagram="saveDiagram"
    >
    </header-component>

    <VueFlow
        :default-edge-options="{ type:'chickenFoot' }"
        @edge-update="onEdgeUpdate"
        @edge-click="openRelationshipModal"
        @connect="onConnect"
        v-model="diagram"
        fit-view-on-init
        :zoomOnDoubleClick = false
        :controlled = false
        class=".vue-flow"
    >
        <!--Chicken foot custom edge component -->
        <template #edge-chickenFoot="props">
            <chicken-foot-edge-component v-bind="props"></chicken-foot-edge-component>
        </template>

        <Background :variant="BackgroundVariant.Lines" />
        <!-- Table -->
        <template #node-table="{ id, data, label }">

            <button class="table_button" @mousedown.stop @click="addRow({ id, data, label })">
                <img class="table_icon" src="../../icons/plus.svg" alt="Add row">
            </button>

            <input class="table_input" v-if="data.editing" :value="label" @input="updateLabel(id, $event.target.value)" @blur="data.editing = false">
            <span class="table_input"  v-else @click="data.editing = true">{{ label }}</span>

            <button class="table_button" @mousedown.stop  @click="deleteNode(id)">
                <img class="table_icon" src="../../icons/cancel.svg" alt="Cancel">
            </button>

        </template>
        <!-- Row -->
        <template #node-row="{ id, data, label }">
            <input
                class="row_input"
                v-if="data.editing"
                :value="label"
                @input="updateLabel(id, $event.target.value)"
                @blur="data.editing = false"
                @keyup.enter="data.editing = false"
            />

            <span class="row_text" v-else @click="data.editing = true">{{ label }}</span>
            <!--SQL Type-->
            <div>
                <select v-model="data.sqlType">
                    <option selected="selected" value="INT(10)">INT</option>
                    <option value="TINYINT(1)">TINYINT</option>
                    <option value="BIGINT">BIGINT</option>
                    <option value="CHAR(255)">CHAR</option>
                    <option value="VARCHAR(255)">VARCHAR</option>
                    <option value="TEXT">TEXT</option>
                    <option value="DATE">DATE</option>
                    <option value="DATETIME">DATETIME</option>
                    <option value="TIME">TIME</option>
                    <option value="TIMESTAMP">TIMESTAMP</option>
                    <option v-bind:value="data.sqlType">{{ data.sqlType }}</option>
                </select>
            </div>

            <!--Options-->
            <button class="table_button" @mousedown.stop @click="toggleOptionsModal(id, $event)">
                <img class="table_icon" src="../../icons/dots.svg" alt="More options">
            </button>

            <!-- Options modal -->
            <div v-if="data.showOptionsModal" class="options_modal" :style="{ left: `${data.modalPosition.x}px`, top: `${data.modalPosition.y}px` }">
                <!--Key mod-->
                <select v-model="data.keyMod" @change="updateKeyMod(id, data.keyMod)">
                    <option selected="selected" value="None">None</option>
                    <option value="Primary">Primary</option>
                    <option value="Unique">Unique</option>
                    <option value="Index">Index</option>
                </select>
                <p class="modal_text">Unsigned</p>
                <input type="checkbox"  @mousedown.stop :checked="data.unsigned" @change="toggleUnsigned(id)">
                <p class="modal_text">Nullable</p>
                <input type="checkbox" @mousedown.stop :checked="data.nullable" @change="toggleNullable(id)">
            </div>
            <!--Delete row-->
            <button class ="table_button" @mousedown.stop  @click="deleteNode(id)">
                <img class="table_icon" src="../../icons/cancel.svg" alt="Cancel">
            </button>

            <Handle type="source" position="right" />
            <Handle type="source" position="left" />

        </template>

    </VueFlow>
    <!--Relationship modal-->
    <div v-if="showRelationshipModal" class="relationship_modal" :style="{ left: `${modalPosition.x}px`, top: `${modalPosition.y}px` }">
        <button @click="updateConnectionLineType('one-to-one')">One to One</button>
        <button @click="updateConnectionLineType('one-to-many')">One to Many</button>
        <button @click="updateConnectionLineType('many-to-many')">Many to Many</button>
        <button @click="deleteEdge">Delete</button>
        <button @click="showRelationshipModal = false">Close</button>
    </div>
    <!--Import modal-->
    <div v-if="showImportModal" class="modal flex-centered">
        <div class="sql_modal_content">
            <textarea class="sql_textarea" v-model="importContent"></textarea>
            <button @click="importSql">Import</button>
            <button @click="showImportModal = false">Close</button>
        </div>
    </div>
    <!--Export modal-->
    <div v-if="showExportModal" class="modal sql_modal flex-centered">
        <div class="sql_modal_content">
            <textarea class="sql_textarea" v-model="exportContent"></textarea>
            <button class="btn btn-primary" @click="exportSql">Export</button>
            <button class="btn btn-primary" @click="showExportModal = false">Close</button>
        </div>
    </div>
    <!--Diagrams modal-->
    <div v-if="showDiagramsModal" class="modal sql_modal flex-centered">
        <div class="sql_modal_content">

            <input v-model="diagramName">
            <button class="btn btn-primary" @mousedown.stop @click="addDiagram">Add</button>

            <div v-for="(diagram, index) in diagrams" :key="index">
                <p @click="selectDiagram(diagram.id, diagram.name)">Name: {{ diagram.name }}</p>
            </div>

            <button class="btn btn-primary" @click="showDiagramsModal = false">Close</button>
        </div>
    </div>

</template>

<script setup>
import { Handle, Position, VueFlow, useVueFlow } from '@vue-flow/core'
import { Background, BackgroundVariant } from '@vue-flow/background'
import {ref, onMounted, onBeforeMount, computed} from 'vue'

import { useStore } from 'vuex';
import { useToast } from 'vue-toast-notification';
const $toast = useToast();
const store = useStore();

const { updateEdge, addEdges } = useVueFlow();
import  { TableActions } from '../services/TableActions.js';
import { ParseSql } from "../services/ParseSql.js";
import {Diagram} from "../services/Diagram";

const modalPosition = ref({ x: 0, y: 0 });
const selectedEdge = ref(null);
const showRelationshipModal = ref(false);

const showImportModal = ref(false);
const importContent = ref('');

const showExportModal = ref(false);
const exportContent = ref('');

const showDiagramsModal = ref(false);

const loggedIn = computed(() => store.getters.loggedIn);
const currentDiagramId = computed(() => store.state.current_diagram_id);

const diagrams = ref([]);
const diagramName = ref('');

const TableStyle = {
  display: 'flex',
  border: '1px solid #10b981',
  background: '#007BFF',
  borderColor: '#007BFF',
  color: 'white',
  borderRadius: '5px',
  width: '350px',
  height: '40px',
  alignItems: 'center',
  justifyContent: 'space-between',
}

const diagram = ref([
  {
    id: '1',
    type: 'table',
    label: 'First table',
    data: { toolbarPosition: Position.Top, toolbarVisible: true },
    position: { x: 0, y: -100 },
    style: TableStyle,
  },
])

const addTable = () => {
  TableActions.addTable(diagram, TableStyle, 'new_table');
}
const addRow = (nodeProps) => {
  TableActions.addRow(diagram, nodeProps, {
    rowName: 'new_row',
    keyMod: 'None',
    sqlType: 'INT(11)',
    nullable: false,
    unsigned: false
  });
};
const deleteEdge = () => {
  TableActions.deleteEdge(diagram, selectedEdge);
};
const deleteNode = (nodeId) => {
  TableActions.deleteNode(diagram, nodeId);
};

const openDiagramsModal = async () => {
    if (loggedIn.value) {
        diagrams.value = await Diagram.getDiagrams();
        showDiagramsModal.value = true
    }
    else {
        $toast.warning('You must login to access diagrams.')
    }
};
const addDiagram = () => {
    Diagram.addDiagram(diagramName.value)
}
const saveDiagram = () => {
    loggedIn.value ? Diagram.saveDiagram(currentDiagramId.value, diagram.value) : $toast.warning('You must login to save diagrams');
}
const selectDiagram = async (id, name) => {
    let selectedDiagram =  await Diagram.selectDiagram(id, name, store);
    console.log(selectedDiagram)
    diagram.value = JSON.parse(selectedDiagram[0].diagram);
}

function onConnect(params) {
    params.updatable = true;
    return addEdges([params])
}

function onEdgeUpdate({ edge, connection }) {
    return updateEdge(edge, connection)
}
const updateConnectionLineType = (relationshipType) => {
  TableActions.updateConnectionLineType(diagram, selectedEdge, relationshipType);
};


const updateLabel = (id, newLabel) => {
  const element = diagram.value.find(el => el.id === id);
  if (element) {
    element.label = newLabel;
  }
}
const updateKeyMod = (id, keyMod) => {
  const element = diagram.value.find(el => el.id === id);
  if (element) {
    element.data.keyMod = keyMod;
  }
}


const toggleNullable = (id) => {
  const element = diagram.value.find(el => el.id=== id);
  if (element) {
    element.data.nullable = !element.data.nullable;
  }
}
const toggleUnsigned = (id) => {
  const element = diagram.value.find(el => el.id === id);
  if (element) {
    element.data.unsigned = !element.data.unsigned;
  }
};


const toggleOptionsModal = (id) => {
  const row = diagram.value.find(el => el.id === id);
  const offsetX = 350;

  const documentX = row.position.x;
  const documentY = row.position.y;

  const rowHeight = 60;
  const rowIndex = diagram.value.find(el => el.id === id);
  const offsetY = rowIndex * (rowHeight-20);

  row.data.modalPosition = { x: documentX + offsetX, y: documentY - offsetY };
  row.data.showOptionsModal = !row.data.showOptionsModal;
};

const openRelationshipModal = (params) => {
  selectedEdge.value = params.edge;
  const edgeElement = document.querySelector(`[id="${params.edge.id}"]`);
  const edgeRect = edgeElement.getBoundingClientRect();
  modalPosition.value = {
    // Calculate the midpoint of the edge
    x: edgeRect.left + window.scrollX + edgeRect.width / 2,
    y: edgeRect.top + window.scrollY + edgeRect.height / 2
  };
  showRelationshipModal.value = true;
};


const openImportModal = () => {
    showImportModal.value = true;
};
const importSql = async () => {
    diagram.value = await ParseSql.importSql(importContent.value);
};


const openExportModal = () => {
    showExportModal.value = true;
};
const exportSql = async  () => {
    exportContent.value = await ParseSql.exportSql(diagram)
}

const loadElementsFromLocalStorage = () => {
    const storedElements = localStorage.getItem('diagram');
    if (storedElements) {
        diagram.value = JSON.parse(storedElements);
    }
}
onBeforeMount(() => {
    loadElementsFromLocalStorage();
})
onMounted(() => {
  setInterval(() => {
    localStorage.setItem('diagram', JSON.stringify(diagram.value));
  }, 5000);
})
</script>

<style scoped>
.sql_modal_content {
  background: white;
  padding: 20px;
  border-radius: 5px;
  height: 500px;
  width: 700px;
}

.sql_textarea {
  width: 100%;
  height: 500px;
  margin-bottom: 10px;
}
.table_button {
  width: 15%;
  height: 80%;
  margin-top: 5px;
  padding: 0;
  border: none;
  background: none;
}
.table_icon {
  width: 70%;
  height: 70%;
  color: white;
}
.row_text{
  width: 150px;
}
.row_input{
  width: 126px;
  height: 5px;
  padding: 10px;
}
.table_input{
  width: 80%;
  padding: 10px;
}
.relationship_modal {
  position: absolute;
  width: 200px;
  padding: 20px;
  background-color: #fff;
  border-radius: 10px;
  box-shadow: 0 0 10px rgba(0, 0, 0, 0.15);
  display: flex;
  flex-direction: column;
  gap: 10px;
}
.relationship_modal button {
  padding: 10px;
  border: none;
  border-radius: 5px;
  background-color: #007BFF;
  color: #fff;
  cursor: pointer;
  transition: background-color 0.3s ease;
}
.relationship_modal button:hover {
  background-color: #0056b3;
}
select {
  padding: 5px;
  border: 1px solid #ccc;
  border-radius: 4px;
  background-color: #f9f9f9;
  color: #333;
  cursor: pointer;

}
select:hover {
  background-color: #f0f0f0;
}
input{
  margin :0;
}
.options_modal{
  position: absolute;
  display: flex;
  align-items: center;
  justify-content: space-between;
  border: 1px solid #007BFF;
  border-radius: 5px;
  width: 300px;
}
.modal_text{
  margin : 0;
  font-size: 15px;

}
</style>
