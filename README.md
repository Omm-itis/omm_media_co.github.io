<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>AI Website Pipeline & Workflow</title>
<script src="https://cdn.tailwindcss.com"></script>
<link href="https://fonts.googleapis.com/css2?family=Inter:wght@300;400;500;600;700&display=swap" rel="stylesheet">
<script>
tailwind.config={theme:{extend:{fontFamily:{sans:['Inter','sans-serif']},colors:{brand:{500:'#f97316',400:'#fb923c',glow:'rgba(249,115,22,0.5)'}}}}}
</script>
<style>
body{margin:0;overflow:hidden;background-color:#f4f4f5;color:#171717;font-family:'Inter',sans-serif;user-select:none;}
.ambient-glow{position:absolute;top:50%;left:50%;width:80vw;height:80vh;transform:translate(-50%,-50%);background:radial-gradient(circle,rgba(249,115,22,0.1) 0%,rgba(255,255,255,0) 70%);pointer-events:none;z-index:0;}
#canvas-container{width:100vw;height:100vh;position:relative;cursor:grab;overflow:hidden;z-index:10;}
#canvas-container:active{cursor:grabbing;}
#canvas-world{position:absolute;transform-origin:0 0;width:0;height:0;transition:transform 0.3s cubic-bezier(0.2,0.8,0.2,1);}
#canvas-world.dragging{transition:none;}
#svg-layer{position:absolute;top:0;left:0;width:10000px;height:10000px;pointer-events:none;z-index:1;overflow:visible;}
.edge-path{fill:none;stroke:rgba(0,0,0,0.1);stroke-width:3px;transition:stroke 0.3s ease,stroke-width 0.3s ease;}
.edge-flow{fill:none;stroke:#f97316;stroke-width:4px;stroke-dasharray:8 24;animation:flow 1.5s linear infinite;filter:drop-shadow(0 0 4px rgba(249,115,22,0.4));opacity:0.8;transition:opacity 0.3s ease,stroke-width 0.3s ease;}
@keyframes flow{from{stroke-dashoffset:32;}to{stroke-dashoffset:0;}}
.edge-group.highlight .edge-path{stroke:rgba(249,115,22,0.3);}
.edge-group.highlight .edge-flow{opacity:1;stroke-width:6px;filter:drop-shadow(0 0 8px rgba(249,115,22,0.6));}
.edge-group.dimmed{opacity:0.2;}
#nodes-layer{position:absolute;top:0;left:0;z-index:2;}
.workflow-node{position:absolute;width:320px;background:rgba(255,255,255,0.75);backdrop-filter:blur(16px);-webkit-backdrop-filter:blur(16px);border:1px solid rgba(0,0,0,0.08);border-radius:20px;padding:24px;box-shadow:0 10px 30px rgba(0,0,0,0.04),inset 0 1px 0 rgba(255,255,255,1);transition:transform 0.3s cubic-bezier(0.175,0.885,0.32,1.275),box-shadow 0.3s ease,border-color 0.3s ease;cursor:pointer;display:flex;flex-direction:column;gap:12px;}
.workflow-node:hover{transform:translateY(-5px) scale(1.02);box-shadow:0 20px 40px rgba(0,0,0,0.08),0 0 20px rgba(249,115,22,0.15);border-color:rgba(249,115,22,0.4);z-index:100;}
.workflow-node.dimmed{opacity:0.4;transform:scale(0.98);}
.node-header{display:flex;justify-content:space-between;align-items:center;}
.node-title{font-size:1.125rem;font-weight:600;color:#111827;letter-spacing:-0.02em;}
.node-desc{font-size:0.875rem;color:#4b5563;line-height:1.5;}
.data-tags{display:flex;flex-wrap:wrap;gap:6px;margin-top:8px;}
.data-tag{background:rgba(0,0,0,0.04);border:1px solid rgba(0,0,0,0.06);color:#374151;font-size:0.7rem;padding:4px 8px;border-radius:6px;font-weight:500;letter-spacing:0.02em;}
.note-indicator{display:flex;align-items:center;gap:6px;font-size:0.75rem;color:#ea580c;margin-top:8px;padding-top:12px;border-top:1px solid rgba(0,0,0,0.05);}
.controls-panel{position:fixed;bottom:30px;left:50%;transform:translateX(-50%);background:rgba(255,255,255,0.85);backdrop-filter:blur(20px);border:1px solid rgba(0,0,0,0.1);padding:8px 16px;border-radius:100px;display:flex;gap:16px;z-index:50;box-shadow:0 10px 30px rgba(0,0,0,0.08);}
.control-btn{background:transparent;border:none;color:#6b7280;cursor:pointer;display:flex;align-items:center;justify-content:center;padding:8px;border-radius:50%;transition:all 0.2s ease;}
.control-btn:hover{color:#111827;background:rgba(0,0,0,0.05);}
#edit-modal{transition:opacity 0.3s ease,visibility 0.3s ease;}
#edit-modal.hidden-modal{opacity:0;visibility:hidden;pointer-events:none;}
.modal-content{transform:scale(0.95) translateY(10px);transition:transform 0.3s cubic-bezier(0.175,0.885,0.32,1.275);}
#edit-modal:not(.hidden-modal) .modal-content{transform:scale(1) translateY(0);}
::-webkit-scrollbar{width:8px;}
::-webkit-scrollbar-track{background:rgba(0,0,0,0.03);border-radius:4px;}
::-webkit-scrollbar-thumb{background:rgba(0,0,0,0.15);border-radius:4px;}
::-webkit-scrollbar-thumb:hover{background:rgba(0,0,0,0.25);}
.control-btn.active{color:#f97316;background:rgba(249,115,22,0.15);}
.linking-source{box-shadow:0 0 0 4px rgba(249,115,22,0.6),0 20px 40px rgba(0,0,0,0.1)!important;border-color:#f97316!important;transform:translateY(-5px) scale(1.02);z-index:100;}
.canvas-linking{cursor:crosshair!important;}
.canvas-linking .workflow-node{cursor:crosshair!important;}
.edge-group:hover .edge-path{stroke:rgba(239,68,68,0.6);stroke-width:6px;}
</style>
</head>
<body class="antialiased">
<div class="ambient-glow"></div>
<div id="canvas-container">
<div id="canvas-world">
<svg id="svg-layer"></svg>
<div id="nodes-layer"></div>
</div>
</div>
<div class="controls-panel">
<button class="control-btn" onclick="addNode()" title="Add Node"><svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><rect x="3" y="3" width="18" height="18" rx="2" ry="2"></rect><line x1="12" y1="8" x2="12" y2="16"></line><line x1="8" y1="12" x2="16" y2="12"></line></svg></button>
<button class="control-btn" id="btn-link" onclick="toggleLinkMode()" title="Link Nodes"><svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M10 13a5 5 0 0 0 7.54.54l3-3a5 5 0 0 0-7.07-7.07l-1.72 1.71"></path><path d="M14 11a5 5 0 0 0-7.54-.54l-3 3a5 5 0 0 0 7.07 7.07l1.71-1.71"></path></svg></button>
<div class="w-px h-6 bg-black/10 my-auto"></div>
<button class="control-btn" onclick="zoomOut()" title="Zoom Out"><svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><circle cx="11" cy="11" r="8"></circle><line x1="21" y1="21" x2="16.65" y2="16.65"></line><line x1="8" y1="11" x2="14" y2="11"></line></svg></button>
<button class="control-btn" onclick="zoomIn()" title="Zoom In"><svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><circle cx="11" cy="11" r="8"></circle><line x1="21" y1="21" x2="16.65" y2="16.65"></line><line x1="11" y1="8" x2="11" y2="14"></line><line x1="8" y1="11" x2="14" y2="11"></line></svg></button>
<div class="w-px h-6 bg-black/10 my-auto"></div>
<button class="control-btn" onclick="fitToScreen()" title="Fit to Screen"><svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M4 14v6h6M20 10V4h-6M10 20H4v-6M14 4h6v6"/></svg></button>
<button class="control-btn" onclick="toggleFullscreen()" title="Fullscreen"><svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M8 3H5a2 2 0 0 0-2 2v3m18 0V5a2 2 0 0 0-2-2h-3m0 18h3a2 2 0 0 0 2-2v-3M3 16v3a2 2 0 0 0 2 2h3"></path></svg></button>
<div class="w-px h-6 bg-black/10 my-auto"></div>
<div class="flex items-center px-2 text-xs text-neutral-500 font-medium">Link: Click nodes. Drag to move. Dbl-click to edit/del.</div>
</div>
<div id="edit-modal" class="fixed inset-0 z-[100] flex items-center justify-center bg-black/20 backdrop-blur-sm hidden-modal">
<div class="modal-content bg-white border border-black/10 rounded-2xl w-full max-w-lg p-6 shadow-2xl flex flex-col gap-5">
<div class="flex justify-between items-center">
<h2 class="text-xl font-semibold text-gray-900">Edit Workflow Node</h2>
<button onclick="closeModal()" class="text-gray-400 hover:text-gray-900 transition-colors"><svg width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><line x1="18" y1="6" x2="6" y2="18"></line><line x1="6" y1="6" x2="18" y2="18"></line></svg></button>
</div>
<input type="hidden" id="modal-node-id">
<div class="flex flex-col gap-2">
<label class="text-xs font-medium text-gray-500 uppercase tracking-wider">Node Title</label>
<input type="text" id="modal-title" class="w-full bg-gray-50 border border-gray-200 rounded-lg px-4 py-3 text-gray-900 focus:outline-none focus:border-brand-500 focus:ring-1 focus:ring-brand-500 transition-all">
</div>
<div class="flex flex-col gap-2">
<label class="text-xs font-medium text-gray-500 uppercase tracking-wider">Description</label>
<textarea id="modal-desc" rows="2" class="w-full bg-gray-50 border border-gray-200 rounded-lg px-4 py-3 text-gray-900 focus:outline-none focus:border-brand-500 focus:ring-1 focus:ring-brand-500 transition-all resize-none"></textarea>
</div>
<div class="flex flex-col gap-2">
<label class="text-xs font-medium text-gray-500 uppercase tracking-wider">Presenter Notes (Private)</label>
<textarea id="modal-notes" rows="4" placeholder="Add personal talking points, meeting notes, or strategies here..." class="w-full bg-orange-50 border border-orange-200 rounded-lg px-4 py-3 text-gray-900 placeholder:text-gray-400 focus:outline-none focus:border-brand-500 focus:ring-1 focus:ring-brand-500 transition-all resize-none"></textarea>
</div>
<div class="flex justify-between items-center mt-2">
<button onclick="deleteNode()" class="px-4 py-2.5 rounded-lg text-sm font-medium text-red-500 hover:text-red-700 hover:bg-red-50 transition-all">Delete Node</button>
<div class="flex gap-3">
<button onclick="closeModal()" class="px-5 py-2.5 rounded-lg text-sm font-medium text-gray-600 hover:text-gray-900 hover:bg-gray-100 transition-all">Cancel</button>
<button onclick="saveModal()" class="px-5 py-2.5 rounded-lg text-sm font-medium bg-brand-500 text-white hover:bg-brand-400 shadow-lg shadow-brand-500/25 transition-all">Save Changes</button>
</div>
</div>
</div>
</div>
<script>
const COL_SPACING=400;const ROW_SPACING=220;
const initialNodes=[
{id:'src_maps',title:'Google Ecosystem',desc:'Sourced from Google Maps, Search, and MyBusiness profiles.',x:0,y:0,fields:['Lead ID','Business Name','City','Google Maps Link']},
{id:'src_social',title:'Social Media',desc:'Discovered via Meta, Instagram, and LinkedIn ads or outreach.',x:0,y:ROW_SPACING,fields:['Lead ID','Business Name','Contact Link']},
{id:'src_offline',title:'Directories & Referrals',desc:'Amazon, JustDial, offline events, and partner referrals.',x:0,y:ROW_SPACING*2,fields:['Lead ID','Business Name','Phone Number']},
{id:'collection',title:'Lead Collection Pool',desc:'Central aggregation. Basic data structuring and initial verification.',x:COL_SPACING,y:ROW_SPACING,fields:['Business Category','Email Address','Area']},
{id:'qual_web',title:'Website Assessment',desc:'Check if they have a website and perform basic technical audit.',x:COL_SPACING*2,y:ROW_SPACING*0.5,fields:['Website?','Website URL','Website Audit Notes']},
{id:'qual_brand',title:'Visual & Brand Audit',desc:'Review quality of product images, brand presentation, and trust markers.',x:COL_SPACING*2,y:ROW_SPACING*1.5,fields:['Google Rating','Review Count','Current Stage']},
{id:'trig_ai_web',title:'AI Website Track Trigger',desc:'Identified: No website, outdated design, poor mobile UX, or slow speeds.',x:COL_SPACING*3,y:ROW_SPACING*0.5,fields:['Potential','Pain Point (Web)']},
{id:'trig_photo',title:'Content Upgrade Trigger',desc:'Identified: Low quality imagery, weak branding, poor trust signals.',x:COL_SPACING*3,y:ROW_SPACING*1.5,fields:['Potential','Pain Point (Brand)']},
{id:'outreach',title:'Outreach & Prioritization',desc:'Initial contact via messaging, sorting leads based on high potential.',x:COL_SPACING*4,y:ROW_SPACING,fields:['WhatsApp Number','Last Contact Date','Call Attempts']},
{id:'calls',title:'Calls & Follow-up',desc:'Active calling phase to establish requirements and pitch initial value.',x:COL_SPACING*5,y:ROW_SPACING,fields:['Next Follow-up Date','Preferred Calling Time','Quick Notes']},
{id:'meeting',title:'Discovery Meeting',desc:'Formal sit-down (virtual or offline) to discuss business gaps.',x:COL_SPACING*6,y:ROW_SPACING,fields:['Meeting Scheduled','Meeting Date','Meeting Time','Meeting Mode']},
{id:'proposal',title:'Requirement & Proposal',desc:'Drafting exact requirements, scope of work, and initial pricing.',x:COL_SPACING*7,y:ROW_SPACING,fields:['Meeting Agenda','Discussion Notes','Client Requirements']},
{id:'negotiate',title:'Pitch & Negotiation',desc:'Handling objections, finalizing scope, and agreeing on price.',x:COL_SPACING*8,y:ROW_SPACING,fields:['Objections Raised','Quoted Price','Final Price']},
{id:'dev',title:'Project Start & Dev',desc:'Development of AI Website or content creation begins.',x:COL_SPACING*9,y:ROW_SPACING,fields:['Project Status','Website Type']},
{id:'revisions',title:'Revisions & Approval',desc:'Iterative feedback loop with client leading to final sign-off.',x:COL_SPACING*10,y:ROW_SPACING,fields:['Revision Count','Delivery Date']},
{id:'invoice',title:'Invoicing',desc:'Final invoice generation and payment collection process.',x:COL_SPACING*11,y:ROW_SPACING,fields:['Invoice Sent','Invoice Number','Payment Method']},
{id:'complete',title:'Project Complete',desc:'Payment secured, project delivered, handover successful.',x:COL_SPACING*12,y:ROW_SPACING,fields:['Payment Status']}
];
let connections=[
{source:'src_maps',target:'collection'},{source:'src_social',target:'collection'},{source:'src_offline',target:'collection'},
{source:'collection',target:'qual_web'},{source:'collection',target:'qual_brand'},{source:'qual_web',target:'trig_ai_web'},
{source:'qual_brand',target:'trig_photo'},{source:'trig_ai_web',target:'outreach'},{source:'trig_photo',target:'outreach'},
{source:'outreach',target:'calls'},{source:'calls',target:'meeting'},{source:'meeting',target:'proposal'},
{source:'proposal',target:'negotiate'},{source:'negotiate',target:'dev'},{source:'dev',target:'revisions'},
{source:'revisions',target:'invoice'},{source:'invoice',target:'complete'}
];
let nodes=[];let transform={x:0,y:0,scale:1};let isDragging=false;let dragStart={x:0,y:0};
let isDraggingNode=false;let draggedNodeId=null;let nodeDragStart={x:0,y:0};let nodeStartPos={x:0,y:0};
let isLinkMode=false;let linkSourceId=null;const NODE_WIDTH=320;const NODE_HEIGHT=180;
function init(){
const savedState=localStorage.getItem('ai_pipeline_state_v2');
if(savedState){try{const parsed=JSON.parse(savedState);nodes=parsed.nodes;connections=parsed.connections;}catch(e){nodes=[...initialNodes];}}else{
const oldState=localStorage.getItem('ai_pipeline_nodes');
if(oldState){try{const parsed=JSON.parse(oldState);nodes=initialNodes.map(baseNode=>{const savedNode=parsed.find(n=>n.id===baseNode.id);return savedNode?{...baseNode,title:savedNode.title,desc:savedNode.desc,notes:savedNode.notes}:baseNode;});}catch(e){nodes=[...initialNodes];}}else{nodes=[...initialNodes];}}
renderGraph();setupInteractions();
setTimeout(()=>{fitToScreen();document.getElementById('canvas-world').style.transition='transform 0.4s cubic-bezier(0.2, 0.8, 0.2, 1)';},100);
}
function saveState(){
const stateToSave={nodes:nodes,connections:connections};localStorage.setItem('ai_pipeline_state_v2',JSON.stringify(stateToSave));
}
function renderGraph(){
const nodesContainer=document.getElementById('nodes-layer');const svgContainer=document.getElementById('svg-layer');
nodesContainer.innerHTML='';svgContainer.innerHTML='';
connections.forEach(conn=>{
const sourceNode=nodes.find(n=>n.id===conn.source);const targetNode=nodes.find(n=>n.id===conn.target);
if(!sourceNode||!targetNode)return;
const startX=sourceNode.x+NODE_WIDTH;const startY=sourceNode.y+80;const endX=targetNode.x;const endY=targetNode.y+80;
const dx=Math.abs(endX-startX);const controlPointOffset=Math.max(dx*0.5,50);
const pathString=`M ${startX} ${startY} C ${startX+controlPointOffset} ${startY}, ${endX-controlPointOffset} ${endY}, ${endX} ${endY}`;
const group=document.createElementNS('http://www.w3.org/2000/svg','g');
group.classList.add('edge-group');group.setAttribute('data-source',conn.source);group.setAttribute('data-target',conn.target);
group.style.pointerEvents='stroke';group.style.cursor='pointer';group.title="Double click to delete connection";
const hitPath=document.createElementNS('http://www.w3.org/2000/svg','path');
hitPath.setAttribute('d',pathString);hitPath.style.fill='none';hitPath.style.stroke='transparent';hitPath.style.strokeWidth='24px';
const path=document.createElementNS('http://www.w3.org/2000/svg','path');
path.setAttribute('d',pathString);path.classList.add('edge-path');
const flowPath=document.createElementNS('http://www.w3.org/2000/svg','path');
flowPath.setAttribute('d',pathString);flowPath.classList.add('edge-flow');
group.appendChild(hitPath);group.appendChild(path);group.appendChild(flowPath);
group.addEventListener('dblclick',(e)=>{
e.stopPropagation();const idx=connections.findIndex(c=>c.source===conn.source&&c.target===conn.target);
if(idx>-1){connections.splice(idx,1);saveState();renderGraph();}
});
svgContainer.appendChild(group);
});
nodes.forEach(node=>{
const el=document.createElement('div');el.className='workflow-node';el.id=`node-${node.id}`;el.style.transform=`translate(${node.x}px, ${node.y}px)`;
let tagsHTML='';if(node.fields&&node.fields.length>0){tagsHTML=`<div class="data-tags">${node.fields.map(f=>`<span class="data-tag">${f}</span>`).join('')}</div>`;}
let notesIndicatorHTML='';if(node.notes&&node.notes.trim()!==''){notesIndicatorHTML=`<div class="note-indicator"><svg width="12" height="12" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M14 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V8z"></path><polyline points="14 2 14 8 20 8"></polyline><line x1="16" y1="13" x2="8" y2="13"></line><line x1="16" y1="17" x2="8" y2="17"></line><polyline points="10 9 9 9 8 9"></polyline></svg>Contains Presenter Notes</div>`;}
el.innerHTML=`<div class="node-header"><h3 class="node-title">${node.title}</h3></div><p class="node-desc">${node.desc}</p>${tagsHTML}${notesIndicatorHTML}`;
if(isLinkMode&&linkSourceId===node.id){el.classList.add('linking-source');}
el.addEventListener('mouseenter',()=>highlightConnections(node.id));
el.addEventListener('mouseleave',clearHighlights);
el.addEventListener('mousedown',(e)=>{
if(isLinkMode)return;e.stopPropagation();isDraggingNode=true;draggedNodeId=node.id;
nodeDragStart={x:e.clientX,y:e.clientY};nodeStartPos={x:node.x,y:node.y};el.style.zIndex='1000';
});
el.addEventListener('touchstart',(e)=>{
if(isLinkMode)return;if(e.touches.length===1){e.stopPropagation();isDraggingNode=true;draggedNodeId=node.id;
nodeDragStart={x:e.touches[0].clientX,y:e.touches[0].clientY};nodeStartPos={x:node.x,y:node.y};el.style.zIndex='1000';}
},{passive:false});
el.addEventListener('click',(e)=>{
if(!isLinkMode)return;e.stopPropagation();
if(!linkSourceId){linkSourceId=node.id;renderGraph();}else{
if(linkSourceId!==node.id){const exists=connections.some(c=>c.source===linkSourceId&&c.target===node.id);if(!exists){connections.push({source:linkSourceId,target:node.id});saveState();}}
linkSourceId=null;renderGraph();}
});
el.addEventListener('dblclick',(e)=>{if(isLinkMode)return;e.stopPropagation();openModal(node.id);});
nodesContainer.appendChild(el);
});
}
function highlightConnections(nodeId){
const allNodes=document.querySelectorAll('.workflow-node');const allEdges=document.querySelectorAll('.edge-group');
const connectedEdges=connections.filter(c=>c.source===nodeId||c.target===nodeId);const connectedNodeIds=new Set();
connectedNodeIds.add(nodeId);connectedEdges.forEach(c=>{connectedNodeIds.add(c.source);connectedNodeIds.add(c.target);});
allNodes.forEach(el=>{const id=el.id.replace('node-','');if(!connectedNodeIds.has(id)){el.classList.add('dimmed');}});
allEdges.forEach(el=>{const source=el.getAttribute('data-source');const target=el.getAttribute('data-target');if(source===nodeId||target===nodeId){el.classList.add('highlight');}else{el.classList.add('dimmed');}});
}
function clearHighlights(){
document.querySelectorAll('.workflow-node').forEach(el=>el.classList.remove('dimmed'));
document.querySelectorAll('.edge-group').forEach(el=>{el.classList.remove('highlight');el.classList.remove('dimmed');});
}
function setupInteractions(){
const container=document.getElementById('canvas-container');const world=document.getElementById('canvas-world');
container.addEventListener('mousedown',(e)=>{
if(e.target.closest('.workflow-node')&&!e.target.closest('#canvas-container'))return;
isDragging=true;dragStart={x:e.clientX-transform.x,y:e.clientY-transform.y};world.classList.add('dragging');
});
window.addEventListener('mousemove',(e)=>{
if(isDraggingNode&&draggedNodeId){
const dx=(e.clientX-nodeDragStart.x)/transform.scale;const dy=(e.clientY-nodeDragStart.y)/transform.scale;
const node=nodes.find(n=>n.id===draggedNodeId);if(node){node.x=nodeStartPos.x+dx;node.y=nodeStartPos.y+dy;renderGraph();}
}else if(isDragging){transform.x=e.clientX-dragStart.x;transform.y=e.clientY-dragStart.y;applyTransform();}
});
window.addEventListener('mouseup',()=>{
if(isDraggingNode){isDraggingNode=false;draggedNodeId=null;saveState();}
isDragging=false;world.classList.remove('dragging');
});
container.addEventListener('touchstart',(e)=>{
if(e.touches.length===1){isDragging=true;dragStart={x:e.touches[0].clientX-transform.x,y:e.touches[0].clientY-transform.y};world.classList.add('dragging');}
},{passive:false});
container.addEventListener('touchmove',(e)=>{
if(isDraggingNode&&draggedNodeId&&e.touches.length===1){e.preventDefault();
const dx=(e.touches[0].clientX-nodeDragStart.x)/transform.scale;const dy=(e.touches[0].clientY-nodeDragStart.y)/transform.scale;
const node=nodes.find(n=>n.id===draggedNodeId);if(node){node.x=nodeStartPos.x+dx;node.y=nodeStartPos.y+dy;renderGraph();}
}else if(isDragging&&e.touches.length===1){e.preventDefault();transform.x=e.touches[0].clientX-dragStart.x;transform.y=e.touches[0].clientY-dragStart.y;applyTransform();}
},{passive:false});
container.addEventListener('touchend',()=>{
if(isDraggingNode){isDraggingNode=false;draggedNodeId=null;saveState();}
isDragging=false;world.classList.remove('dragging');
});
container.addEventListener('wheel',(e)=>{
e.preventDefault();const zoomIntensity=0.001;const delta=-e.deltaY*zoomIntensity;
let newScale=transform.scale*Math.exp(delta);newScale=Math.max(0.1,Math.min(newScale,3));
const rect=container.getBoundingClientRect();const mouseX=e.clientX-rect.left;const mouseY=e.clientY-rect.top;
transform.x=mouseX-(mouseX-transform.x)*(newScale/transform.scale);transform.y=mouseY-(mouseY-transform.y)*(newScale/transform.scale);
transform.scale=newScale;applyTransform();
},{passive:false});
}
function applyTransform(){document.getElementById('canvas-world').style.transform=`translate(${transform.x}px, ${transform.y}px) scale(${transform.scale})`;}
function zoomIn(){transform.scale=Math.min(transform.scale*1.2,3);applyTransform();}
function zoomOut(){transform.scale=Math.max(transform.scale/1.2,0.1);applyTransform();}
function fitToScreen(){
let minX=Infinity,minY=Infinity,maxX=-Infinity,maxY=-Infinity;
nodes.forEach(n=>{if(n.x<minX)minX=n.x;if(n.y<minY)minY=n.y;if(n.x>maxX)maxX=n.x;if(n.y>maxY)maxY=n.y;});
maxX+=NODE_WIDTH;maxY+=NODE_HEIGHT;const padding=100;const contentWidth=maxX-minX;const contentHeight=maxY-minY;
const screenWidth=window.innerWidth;const screenHeight=window.innerHeight;
const scaleX=(screenWidth-padding*2)/contentWidth;const scaleY=(screenHeight-padding*2)/contentHeight;
const scale=Math.min(scaleX,scaleY,1.2);
const targetX=(screenWidth-contentWidth*scale)/2-(minX*scale);const targetY=(screenHeight-contentHeight*scale)/2-(minY*scale);
transform={x:targetX,y:targetY,scale:scale};applyTransform();
}
function toggleFullscreen(){
if(!document.fullscreenElement){document.documentElement.requestFullscreen().catch(err=>{console.log(`Error: ${err.message}`);});}
else{if(document.exitFullscreen){document.exitFullscreen();}}
}
function addNode(){
const screenWidth=window.innerWidth;const screenHeight=window.innerHeight;
const centerX=(screenWidth/2-transform.x)/transform.scale;const centerY=(screenHeight/2-transform.y)/transform.scale;
const newNode={id:'node_'+Date.now(),title:'New Node',desc:'Double click to edit details.',x:centerX-(NODE_WIDTH/2),y:centerY-90,fields:[],notes:''};
nodes.push(newNode);saveState();renderGraph();setTimeout(()=>openModal(newNode.id),50);
}
function toggleLinkMode(){
isLinkMode=!isLinkMode;linkSourceId=null;const btn=document.getElementById('btn-link');const container=document.getElementById('canvas-container');
if(isLinkMode){btn.classList.add('active');container.classList.add('canvas-linking');}else{btn.classList.remove('active');container.classList.remove('canvas-linking');}
renderGraph();
}
function deleteNode(){
const id=document.getElementById('modal-node-id').value;nodes=nodes.filter(n=>n.id!==id);
connections=connections.filter(c=>c.source!==id&&c.target!==id);saveState();renderGraph();closeModal();
}
function openModal(nodeId){
const node=nodes.find(n=>n.id===nodeId);if(!node)return;
document.getElementById('modal-node-id').value=node.id;document.getElementById('modal-title').value=node.title;
document.getElementById('modal-desc').value=node.desc;document.getElementById('modal-notes').value=node.notes||'';
const modal=document.getElementById('edit-modal');modal.classList.remove('hidden-modal');
setTimeout(()=>document.getElementById('modal-title').focus(),100);
}
function closeModal(){const modal=document.getElementById('edit-modal');modal.classList.add('hidden-modal');}
function saveModal(){
const id=document.getElementById('modal-node-id').value;const title=document.getElementById('modal-title').value.trim();
const desc=document.getElementById('modal-desc').value.trim();const notes=document.getElementById('modal-notes').value.trim();
const nodeIndex=nodes.findIndex(n=>n.id===id);
if(nodeIndex>-1){nodes[nodeIndex].title=title||'Untitled Node';nodes[nodeIndex].desc=desc;nodes[nodeIndex].notes=notes;saveState();renderGraph();}
closeModal();
}
window.addEventListener('keydown',(e)=>{if(e.key==='Escape'){if(isLinkMode)toggleLinkMode();closeModal();}});
document.getElementById('edit-modal').addEventListener('mousedown',(e)=>{if(e.target===document.getElementById('edit-modal')){closeModal();}});
window.onload=init;
</script>
</body>
</html>
