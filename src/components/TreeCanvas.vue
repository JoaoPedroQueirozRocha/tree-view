<template>
	<div class="tree-canvas-container">
		<svg
			ref="svgRef"
			class="tree-canvas"
			:width="canvasWidth"
			:height="canvasHeight"
			@mousedown="handleCanvasMouseDown"
			@mousemove="handleCanvasMouseMove"
			@mouseup="handleCanvasMouseUp"
			@touchstart="handleCanvasTouchStart"
			@touchmove="handleCanvasTouchMove"
			@touchend="handleCanvasTouchEnd"
			@dblclick="handleCanvasDoubleClick"
		>
			<!-- Conexões entre nós -->
			<g class="connections">
				<line
					v-for="connection in renderConnections"
					:key="connection.id"
					:x1="connection.x1"
					:y1="connection.y1"
					:x2="connection.x2"
					:y2="connection.y2"
					:class="['connection-line', connection.type]"
					:stroke="connection.type === 'left' ? '#4CAF50' : '#2196F3'"
					stroke-width="3"
					@click="handleConnectionClick(connection)"
				/>
			</g>

			<!-- Nós da árvore -->
			<g class="nodes">
				<g
					v-for="node in Array.from(nodes.values())"
					:key="node.id"
					:transform="`translate(${node.x}, ${node.y})`"
					:class="[
						'node',
						{
							dragging: dragState.nodeId === node.id,
							selected: selectedNode === node.id,
							connecting: connectingState.fromNode === node.id,
						},
					]"
					@mousedown="handleNodeMouseDown($event, node)"
					@touchstart="handleNodeTouchStart($event, node)"
				>
					<!-- Círculo do nó -->
					<circle
						:r="nodeRadius"
						:fill="getNodeColor(node)"
						:stroke="getNodeStroke(node)"
						stroke-width="3"
						class="node-circle"
					/>

					<!-- Área de toque para o texto (invisível, maior) -->
					<circle
						:r="nodeRadius * 0.8"
						fill="transparent"
						class="text-touch-area"
						@touchstart="handleNodeTextTouchStart($event, node)"
						@touchend="handleNodeTextTouchEnd($event, node)"
						v-if="!props.isLinkMode"
					/>

					<!-- Valor do nó -->
					<text
						:x="0"
						:y="5"
						text-anchor="middle"
						class="node-text"
						@click="handleNodeTextClick(node)"
						@touchstart="handleNodeTextTouchStart($event, node)"
						@touchend="handleNodeTextTouchEnd($event, node)"
						:style="{ cursor: props.isLinkMode ? 'default' : 'text' }"
						:class="{ editable: !props.isLinkMode }"
					>
						{{ node.value }}
					</text>

					<!-- Pontos de conexão -->
					<g v-if="isLinkMode" class="connection-points">
						<!-- Ponto esquerdo -->
						<circle
							:cx="-nodeRadius * 0.7"
							:cy="nodeRadius * 0.7"
							r="8"
							class="connection-point left-point"
							fill="#4CAF50"
							@click="handleConnectionPointClick($event, node, 'left')"
						/>
						<!-- Ponto direito -->
						<circle
							:cx="nodeRadius * 0.7"
							:cy="nodeRadius * 0.7"
							r="8"
							class="connection-point right-point"
							fill="#2196F3"
							@click="handleConnectionPointClick($event, node, 'right')"
						/>
					</g>

					<!-- Botão de deletar -->
					<g v-if="selectedNode === node.id && !isLinkMode" class="delete-button">
						<!-- Área de toque invisível maior -->
						<circle
							:cx="nodeRadius * 0.8"
							:cy="-nodeRadius * 0.8"
							r="20"
							fill="transparent"
							class="delete-touch-area"
							@click="handleDeleteClick($event, node)"
							@touchstart="handleDeleteTouchStart($event, node)"
							@touchend="handleDeleteTouchEnd($event, node)"
						/>
						<!-- Círculo visível -->
						<circle
							:cx="nodeRadius * 0.8"
							:cy="-nodeRadius * 0.8"
							r="12"
							fill="#f44336"
							class="delete-circle"
							@click="handleDeleteClick($event, node)"
							@touchstart="handleDeleteTouchStart($event, node)"
							@touchend="handleDeleteTouchEnd($event, node)"
						/>
						<text
							:x="nodeRadius * 0.8"
							:y="-nodeRadius * 0.8 + 4"
							text-anchor="middle"
							fill="white"
							font-size="14"
							font-weight="bold"
							class="delete-text"
							@click="handleDeleteClick($event, node)"
							@touchstart="handleDeleteTouchStart($event, node)"
							@touchend="handleDeleteTouchEnd($event, node)"
						>
							×
						</text>
					</g>
				</g>
			</g>

			<!-- Linha de conexão temporária -->
			<line
				v-if="connectingState.isConnecting"
				:x1="connectingState.startX"
				:y1="connectingState.startY"
				:x2="connectingState.currentX"
				:y2="connectingState.currentY"
				stroke="#FF9800"
				stroke-width="2"
				stroke-dasharray="5,5"
				class="temp-connection"
			/>
		</svg>

		<!-- Input para editar valor do nó -->
		<input
			v-if="editingNode"
			ref="editInput"
			v-model="editingValue"
			type="text"
			class="node-edit-input"
			:style="{
				left: editingNode.x + 'px',
				top: editingNode.y + 'px',
			}"
			@blur="finishEditing"
			@keyup.enter="finishEditing"
			@keyup.escape="cancelEditing"
		/>
	</div>
</template>

<script setup>
import { ref, reactive, computed, onMounted, onUnmounted, nextTick } from 'vue';

// Props
const props = defineProps({
	nodes: {
		type: Map,
		required: true,
	},
	connections: {
		type: Map,
		required: true,
	},
	isLinkMode: {
		type: Boolean,
		default: false,
	},
});

// Emits
const emit = defineEmits(['add-node', 'move-node', 'delete-node', 'connect-nodes', 'disconnect-nodes']);

// Refs
const svgRef = ref(null);
const editInput = ref(null);

// Estado do canvas
const canvasWidth = ref(800);
const canvasHeight = ref(600);
const nodeRadius = 25;

// Estado de interação
const selectedNode = ref(null);
const dragState = reactive({
	isDragging: false,
	nodeId: null,
	startX: 0,
	startY: 0,
	offsetX: 0,
	offsetY: 0,
});

// Estado para controle de toque
const touchState = reactive({
	startTime: 0,
	isLongPress: false,
	touchStarted: false,
});

const connectingState = reactive({
	isConnecting: false,
	fromNode: null,
	fromPosition: null,
	startX: 0,
	startY: 0,
	currentX: 0,
	currentY: 0,
});

// Estado de edição
const editingNode = ref(null);
const editingValue = ref('');

// Computed
const renderConnections = computed(() => {
	const connections = [];

	for (const node of props.nodes.values()) {
		if (node.leftChild) {
			const child = props.nodes.get(node.leftChild);
			if (child) {
				connections.push({
					id: `${node.id}-left-${child.id}`,
					x1: node.x,
					y1: node.y,
					x2: child.x,
					y2: child.y,
					type: 'left',
					parent: node.id,
					child: child.id,
				});
			}
		}

		if (node.rightChild) {
			const child = props.nodes.get(node.rightChild);
			if (child) {
				connections.push({
					id: `${node.id}-right-${child.id}`,
					x1: node.x,
					y1: node.y,
					x2: child.x,
					y2: child.y,
					type: 'right',
					parent: node.id,
					child: child.id,
				});
			}
		}
	}

	return connections;
});

// Métodos utilitários
const getNodeColor = (node) => {
	if (selectedNode.value === node.id) return '#FFE0B2';
	if (connectingState.fromNode === node.id) return '#FFCDD2';
	return '#E3F2FD';
};

const getNodeStroke = (node) => {
	if (selectedNode.value === node.id) return '#FF9800';
	if (connectingState.fromNode === node.id) return '#f44336';
	return '#2196F3';
};

const getMousePosition = (event) => {
	const rect = svgRef.value.getBoundingClientRect();
	return {
		x: event.clientX - rect.left,
		y: event.clientY - rect.top,
	};
};

const getTouchPosition = (event) => {
	const rect = svgRef.value.getBoundingClientRect();
	const touch = event.touches[0] || event.changedTouches[0];
	return {
		x: touch.clientX - rect.left,
		y: touch.clientY - rect.top,
	};
};

const findNodeAtPosition = (x, y) => {
	for (const node of props.nodes.values()) {
		const distance = Math.sqrt((x - node.x) ** 2 + (y - node.y) ** 2);
		if (distance <= nodeRadius) {
			return node;
		}
	}
	return null;
};

// Handlers de mouse
const handleCanvasMouseDown = (event) => {
	if (event.target === svgRef.value) {
		selectedNode.value = null;
	}
};

const handleCanvasMouseMove = (event) => {
	if (dragState.isDragging) {
		const pos = getMousePosition(event);
		const newX = pos.x - dragState.offsetX;
		const newY = pos.y - dragState.offsetY;

		emit('move-node', dragState.nodeId, { x: newX, y: newY });
	}

	if (connectingState.isConnecting) {
		const pos = getMousePosition(event);
		connectingState.currentX = pos.x;
		connectingState.currentY = pos.y;
	}
};

const handleCanvasMouseUp = (event) => {
	if (connectingState.isConnecting) {
		const pos = getMousePosition(event);
		const targetNode = findNodeAtPosition(pos.x, pos.y);

		if (targetNode && targetNode.id !== connectingState.fromNode) {
			emit('connect-nodes', connectingState.fromNode, targetNode.id, connectingState.fromPosition);
		}

		connectingState.isConnecting = false;
		connectingState.fromNode = null;
		connectingState.fromPosition = null;
	}

	dragState.isDragging = false;
	dragState.nodeId = null;
};

const handleCanvasDoubleClick = (event) => {
	if (event.target === svgRef.value) {
		const pos = getMousePosition(event);
		emit('add-node', pos);
	}
};

// Handlers de touch
const handleCanvasTouchStart = (event) => {
	event.preventDefault();
	if (event.target === svgRef.value) {
		selectedNode.value = null;
	}
};

const handleCanvasTouchMove = (event) => {
	event.preventDefault();

	if (dragState.isDragging) {
		const pos = getTouchPosition(event);
		const newX = pos.x - dragState.offsetX;
		const newY = pos.y - dragState.offsetY;

		emit('move-node', dragState.nodeId, { x: newX, y: newY });
	}

	if (connectingState.isConnecting) {
		const pos = getTouchPosition(event);
		connectingState.currentX = pos.x;
		connectingState.currentY = pos.y;
	}
};

const handleCanvasTouchEnd = (event) => {
	event.preventDefault();

	if (connectingState.isConnecting) {
		const pos = getTouchPosition(event);
		const targetNode = findNodeAtPosition(pos.x, pos.y);

		if (targetNode && targetNode.id !== connectingState.fromNode) {
			emit('connect-nodes', connectingState.fromNode, targetNode.id, connectingState.fromPosition);
		}

		connectingState.isConnecting = false;
		connectingState.fromNode = null;
		connectingState.fromPosition = null;
	}

	dragState.isDragging = false;
	dragState.nodeId = null;
};

// Handlers de nó
const handleNodeMouseDown = (event, node) => {
	event.stopPropagation();

	if (props.isLinkMode) return;

	selectedNode.value = node.id;

	const pos = getMousePosition(event);
	dragState.isDragging = true;
	dragState.nodeId = node.id;
	dragState.startX = pos.x;
	dragState.startY = pos.y;
	dragState.offsetX = pos.x - node.x;
	dragState.offsetY = pos.y - node.y;
};

const handleNodeTouchStart = (event, node) => {
	event.preventDefault();
	event.stopPropagation();

	if (props.isLinkMode) return;

	// Se o toque foi no texto, não inicia o drag
	if (event.target.tagName === 'text') {
		return;
	}

	selectedNode.value = node.id;

	const pos = getTouchPosition(event);
	dragState.isDragging = true;
	dragState.nodeId = node.id;
	dragState.startX = pos.x;
	dragState.startY = pos.y;
	dragState.offsetX = pos.x - node.x;
	dragState.offsetY = pos.y - node.y;

	touchState.isLongPress = true;
};

const handleNodeTextClick = (node) => {
	if (!props.isLinkMode) {
		startEditing(node);
	}
};

const handleNodeTextTouchStart = (event, node) => {
	event.preventDefault();
	event.stopPropagation();

	if (props.isLinkMode) return;

	touchState.startTime = Date.now();
	touchState.touchStarted = true;
	touchState.isLongPress = false;

	selectedNode.value = node.id;
};

const handleNodeTextTouchEnd = (event, node) => {
	event.preventDefault();
	event.stopPropagation();

	if (props.isLinkMode) return;
	if (!touchState.touchStarted) return;

	const touchDuration = Date.now() - touchState.startTime;
	touchState.touchStarted = false;

	// Se foi um toque rápido (menos de 300ms), inicia edição
	if (touchDuration < 300 && !touchState.isLongPress) {
		startEditing(node);
	}
};

const handleConnectionPointClick = (event, node, position) => {
	event.stopPropagation();

	if (!props.isLinkMode) return;

	connectingState.isConnecting = true;
	connectingState.fromNode = node.id;
	connectingState.fromPosition = position;
	connectingState.startX = node.x;
	connectingState.startY = node.y;
	connectingState.currentX = node.x;
	connectingState.currentY = node.y;
};

const handleDeleteClick = (event, node) => {
	event.stopPropagation();
	emit('delete-node', node.id);
	selectedNode.value = null;
};

const handleDeleteTouchStart = (event, node) => {
	event.preventDefault();
	event.stopPropagation();
};

const handleDeleteTouchEnd = (event, node) => {
	event.preventDefault();
	event.stopPropagation();
	emit('delete-node', node.id);
	selectedNode.value = null;
};

const handleConnectionClick = (connection) => {
	if (props.isLinkMode) {
		emit('disconnect-nodes', connection.parent, connection.child);
	}
};

// Edição de nós
const startEditing = (node) => {
	editingNode.value = node;
	editingValue.value = node.value.toString();

	nextTick(() => {
		if (editInput.value) {
			editInput.value.focus();
			editInput.value.select();
		}
	});
};

const finishEditing = () => {
	if (editingNode.value) {
		const newValue = editingValue.value.trim();
		if (newValue) {
			editingNode.value.value = newValue;
		}
	}
	cancelEditing();
};

const cancelEditing = () => {
	editingNode.value = null;
	editingValue.value = '';
};

// Redimensionamento
const updateCanvasSize = () => {
	if (svgRef.value) {
		const container = svgRef.value.parentElement;
		canvasWidth.value = container.clientWidth;
		canvasHeight.value = container.clientHeight;
	}
};

// Lifecycle
onMounted(() => {
	updateCanvasSize();
	window.addEventListener('resize', updateCanvasSize);
});

onUnmounted(() => {
	window.removeEventListener('resize', updateCanvasSize);
});
</script>

<style scoped>
.tree-canvas-container {
	width: 100%;
	height: 100%;
	position: relative;
	background: linear-gradient(45deg, #f0f0f0 25%, transparent 25%),
		linear-gradient(-45deg, #f0f0f0 25%, transparent 25%), linear-gradient(45deg, transparent 75%, #f0f0f0 75%),
		linear-gradient(-45deg, transparent 75%, #f0f0f0 75%);
	background-size: 20px 20px;
	background-position: 0 0, 0 10px, 10px -10px, -10px 0px;
}

.tree-canvas {
	width: 100%;
	height: 100%;
	cursor: crosshair;
}

.node {
	cursor: pointer;
	transition: all 0.2s ease;
}

.node:hover .node-circle {
	filter: drop-shadow(0 4px 8px rgba(0, 0, 0, 0.2));
}

.node.dragging {
	cursor: grabbing;
}

.node.selected .node-circle {
	filter: drop-shadow(0 4px 12px rgba(255, 152, 0, 0.5));
}

.node.connecting .node-circle {
	animation: pulse 1s infinite;
}

@keyframes pulse {
	0% {
		transform: scale(1);
	}
	50% {
		transform: scale(1.1);
	}
	100% {
		transform: scale(1);
	}
}

.node-circle {
	transition: all 0.2s ease;
}

.node-text {
	font-size: 16px;
	font-weight: bold;
	fill: #1a1a1a !important;
	color: #1a1a1a !important;
	pointer-events: auto;
	user-select: none;
	cursor: text;
	transition: all 0.2s ease;
}

.node-text:hover {
	fill: #0d47a1 !important;
	color: #0d47a1 !important;
}

.node-text.editable:hover {
	fill: #1976d2 !important;
	color: #1976d2 !important;
	text-shadow: 0 1px 3px rgba(255, 255, 255, 0.9);
}

.connection-line {
	cursor: pointer;
	transition: all 0.2s ease;
}

.connection-line:hover {
	stroke-width: 5;
	filter: drop-shadow(0 2px 4px rgba(0, 0, 0, 0.3));
}

.connection-point {
	cursor: pointer;
	transition: all 0.2s ease;
	opacity: 0.8;
}

.connection-point:hover {
	opacity: 1;
	transform: scale(1.2);
}

.delete-button {
	cursor: pointer;
	animation: fadeInScale 0.3s ease-out;
}

@keyframes fadeInScale {
	0% {
		opacity: 0;
		transform: scale(0.5);
	}
	100% {
		opacity: 1;
		transform: scale(1);
	}
}

.delete-touch-area {
	cursor: pointer;
}

.delete-circle {
	transition: all 0.2s ease;
}

.delete-circle:hover {
	transform: scale(1.1);
	filter: drop-shadow(0 2px 4px rgba(0, 0, 0, 0.3));
}

.delete-text {
	pointer-events: none;
	user-select: none;
	fill: white !important;
	color: white !important;
}

.temp-connection {
	pointer-events: none;
}

.node-edit-input {
	position: absolute;
	transform: translate(-50%, -50%);
	width: 80px;
	height: 35px;
	border: 3px solid #2196f3;
	border-radius: 6px;
	text-align: center;
	font-size: 16px;
	font-weight: bold;
	background: white;
	color: #1a1a1a;
	z-index: 1000;
	box-shadow: 0 4px 12px rgba(0, 0, 0, 0.2);
}

.node-edit-input:focus {
	outline: none;
	border-color: #1976d2;
	box-shadow: 0 0 0 3px rgba(33, 150, 243, 0.3), 0 4px 12px rgba(0, 0, 0, 0.2);
}

/* Responsividade para tablets */
@media (max-width: 768px) {
	.tree-canvas {
		cursor: default;
	}

	.node-text {
		font-size: 18px;
		fill: #1a1a1a !important;
		color: #1a1a1a !important;
	}

	.text-touch-area {
		cursor: pointer;
	}

	.node.selected .text-touch-area {
		fill: rgba(33, 150, 243, 0.1);
		stroke: #2196f3;
		stroke-width: 2;
		stroke-dasharray: 3, 3;
	}

	.connection-point {
		r: 12;
	}

	.delete-circle {
		r: 18;
	}

	.delete-text {
		font-size: 20px;
		fill: white !important;
		color: white !important;
	}

	.node-edit-input {
		width: 90px;
		height: 40px;
		font-size: 18px;
		border-width: 4px;
	}

	.delete-button {
		cursor: pointer;
	}
}

/* Suporte para touch */
@media (hover: none) and (pointer: coarse) {
	.node:hover .node-circle {
		filter: none;
	}

	.connection-line:hover {
		stroke-width: 3;
		filter: none;
	}

	.connection-point:hover {
		opacity: 0.8;
		transform: none;
	}

	.delete-circle:hover {
		transform: none;
		filter: none;
	}
}

.text-touch-area {
	cursor: text;
	pointer-events: auto;
}
</style>
