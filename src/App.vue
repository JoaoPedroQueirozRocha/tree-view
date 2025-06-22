<template>
	<div id="app">
		<header class="header">
			<h1>Editor de Árvores Binárias</h1>
			<div class="toolbar">
				<button @click="addNode" class="btn btn-primary">
					<span class="icon">+</span>
					Adicionar Nó
				</button>
				<button @click="clearCanvas" class="btn btn-danger">
					<span class="icon">🗑️</span>
					Limpar
				</button>
				<button @click="toggleMode" class="btn btn-secondary">
					<span class="icon">🔗</span>
					{{ isLinkMode ? 'Modo Mover' : 'Modo Ligar' }}
				</button>
			</div>
			<div class="help-text">
				<span v-if="!isLinkMode">
					💡 Toque rápido no centro do nó para editar texto • Toque longo para mover • Toque no ❌ para
					deletar
				</span>
				<span v-else> 💡 Use os pontos coloridos para conectar nós • Verde = esquerda, Azul = direita </span>
			</div>
		</header>

		<main class="main-content">
			<TreeCanvas
				:nodes="nodes"
				:connections="connections"
				:is-link-mode="isLinkMode"
				@add-node="handleAddNode"
				@move-node="handleMoveNode"
				@delete-node="handleDeleteNode"
				@connect-nodes="handleConnectNodes"
				@disconnect-nodes="handleDisconnectNodes"
			/>
		</main>
	</div>
</template>

<script setup>
import { ref, reactive } from 'vue';
import TreeCanvas from './components/TreeCanvas.vue';

// Estado da aplicação
const nodes = reactive(new Map());
const connections = reactive(new Map());
const isLinkMode = ref(false);
let nodeIdCounter = 1;

// Adicionar um novo nó
const addNode = () => {
	const id = `node-${nodeIdCounter++}`;
	const node = {
		id,
		x: Math.random() * 400 + 100,
		y: Math.random() * 300 + 100,
		value: nodeIdCounter - 1,
		leftChild: null,
		rightChild: null,
		parent: null,
	};
	nodes.set(id, node);
};

// Handlers para eventos do canvas
const handleAddNode = (position) => {
	const id = `node-${nodeIdCounter++}`;
	const node = {
		id,
		x: position.x,
		y: position.y,
		value: nodeIdCounter - 1,
		leftChild: null,
		rightChild: null,
		parent: null,
	};
	nodes.set(id, node);
};

const handleMoveNode = (nodeId, position) => {
	const node = nodes.get(nodeId);
	if (node) {
		node.x = position.x;
		node.y = position.y;
	}
};

const handleDeleteNode = (nodeId) => {
	const node = nodes.get(nodeId);
	if (node) {
		// Remove conexões
		if (node.parent) {
			const parent = nodes.get(node.parent);
			if (parent.leftChild === nodeId) parent.leftChild = null;
			if (parent.rightChild === nodeId) parent.rightChild = null;
		}

		// Remove filhos
		if (node.leftChild) {
			const leftChild = nodes.get(node.leftChild);
			if (leftChild) leftChild.parent = null;
		}
		if (node.rightChild) {
			const rightChild = nodes.get(node.rightChild);
			if (rightChild) rightChild.parent = null;
		}

		nodes.delete(nodeId);
	}
};

const handleConnectNodes = (parentId, childId, position) => {
	const parent = nodes.get(parentId);
	const child = nodes.get(childId);

	if (parent && child && parentId !== childId) {
		// Remove conexão anterior do filho
		if (child.parent) {
			const oldParent = nodes.get(child.parent);
			if (oldParent.leftChild === childId) oldParent.leftChild = null;
			if (oldParent.rightChild === childId) oldParent.rightChild = null;
		}

		// Adiciona nova conexão
		if (position === 'left') {
			if (parent.leftChild) {
				const oldChild = nodes.get(parent.leftChild);
				if (oldChild) oldChild.parent = null;
			}
			parent.leftChild = childId;
		} else {
			if (parent.rightChild) {
				const oldChild = nodes.get(parent.rightChild);
				if (oldChild) oldChild.parent = null;
			}
			parent.rightChild = childId;
		}

		child.parent = parentId;
	}
};

const handleDisconnectNodes = (parentId, childId) => {
	const parent = nodes.get(parentId);
	const child = nodes.get(childId);

	if (parent && child) {
		if (parent.leftChild === childId) parent.leftChild = null;
		if (parent.rightChild === childId) parent.rightChild = null;
		child.parent = null;
	}
};

const clearCanvas = () => {
	nodes.clear();
	connections.clear();
	nodeIdCounter = 1;
};

const toggleMode = () => {
	isLinkMode.value = !isLinkMode.value;
};

// Adicionar alguns nós iniciais
addNode();
addNode();
</script>

<style scoped>
#app {
	height: 100vh;
	display: flex;
	flex-direction: column;
	font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
}

.header {
	background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
	color: white;
	padding: 1rem 2rem;
	box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
}

.header h1 {
	margin: 0 0 1rem 0;
	font-size: 1.5rem;
	font-weight: 600;
}

.toolbar {
	display: flex;
	gap: 1rem;
	flex-wrap: wrap;
}

.btn {
	display: flex;
	align-items: center;
	gap: 0.5rem;
	padding: 0.75rem 1.5rem;
	border: none;
	border-radius: 8px;
	font-size: 0.9rem;
	font-weight: 500;
	cursor: pointer;
	transition: all 0.2s ease;
	user-select: none;
}

.btn:hover {
	transform: translateY(-2px);
	box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
}

.btn:active {
	transform: translateY(0);
}

.btn-primary {
	background: #4caf50;
	color: white;
}

.btn-primary:hover {
	background: #45a049;
}

.btn-danger {
	background: #f44336;
	color: white;
}

.btn-danger:hover {
	background: #da190b;
}

.btn-secondary {
	background: #2196f3;
	color: white;
}

.btn-secondary:hover {
	background: #1976d2;
}

.icon {
	font-size: 1.1rem;
}

.main-content {
	flex: 1;
	overflow: hidden;
}

.help-text {
	margin-top: 0.75rem;
	font-size: 0.85rem;
	color: rgba(255, 255, 255, 0.9);
	font-style: italic;
}

@media (max-width: 768px) {
	.header {
		padding: 1rem;
	}

	.header h1 {
		font-size: 1.25rem;
	}

	.toolbar {
		gap: 0.5rem;
	}

	.btn {
		padding: 0.5rem 1rem;
		font-size: 0.8rem;
	}

	.btn .icon {
		font-size: 1rem;
	}
}
</style>
