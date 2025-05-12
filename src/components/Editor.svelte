<script lang="ts">
	import Konva from 'konva';
	import type { Stage } from 'konva/lib/Stage';
	import type { Layer } from 'konva/lib/Layer';
	import type { Group } from 'konva/lib/Group';
	import type { Circle } from 'konva/lib/shapes/Circle';

	import { onMount, onDestroy, tick } from 'svelte';
	import { writable, derived } from 'svelte/store';

	let stage: Stage | null = null;
	let textCounter = 0;
	const availableFonts = writable(['Arial', 'Verdana', 'Times New Roman', 'sans-serif']);
	let selectedFontFamilyForNewText = 'Arial';

	onMount(async () => {
		await tick(); // Ensure DOM is ready

		const containerElement = document.getElementById('container');
		if (!containerElement) {
			console.error("Container element not found for Konva stage.");
			return;
		}

		stage = new Konva.Stage({
			container: 'container',
			width: containerElement.offsetWidth || 595,
			height: window.innerHeight * 0.8
		});

		const initialLayer = new Konva.Layer();
		pagesStore.set([initialLayer]);
		currentPageIndexStore.set(0);
		// The reactive block above will add initialLayer to stage.

		const handleResize = () => {
			if (stage && browser) {
				const container = document.getElementById('container');
				if (container) {
					stage.width(container.offsetWidth);
					stage.height(window.innerHeight * 0.8);
				}
			}
		};

		window.addEventListener('resize', handleResize);
		handleResize(); // Initial call

		return () => {
			window.removeEventListener('resize', handleResize);
			if (stage) {
				stage.destroy();
				stage = null;
			}
		};
	});

	// --- Pagination Functions ---
	function addBlankPage() {
		const newLayer = new Konva.Layer();
		pagesStore.update(currentPages => {
			const updatedPages = [...currentPages, newLayer];
			currentPageIndexStore.set(updatedPages.length - 1);
			return updatedPages;
		});
	}

	function nextPage() {
		if ($currentPageIndexStore < $pagesStore.length - 1) {
			currentPageIndexStore.update(n => n + 1);
		}
	}

	function prevPage() {
		if ($currentPageIndexStore > 0) {
			currentPageIndexStore.update(n => n - 1);
		}
	}
	// --- End Pagination Functions ---


	function update(activeAnchor: Circle, evt: any) {
		const group = activeAnchor.getParent() as Group;
		const topLeft = group.findOne('.topLeft') as Circle;
		const topRight = group.findOne('.topRight') as Circle;
		const bottomRight = group.findOne('.bottomRight') as Circle;
		const bottomLeft = group.findOne('.bottomLeft') as Circle;
		
		const imageNode = group.findOne(n => n.getClassName() === 'Image');
		const textNode = group.findOne(n => n.getClassName() === 'Text' && n.name() === 'textContent');
		const contentNode = imageNode || textNode;

		let anchorX = activeAnchor.x();
		let anchorY = activeAnchor.y();

		if (evt && evt.evt.shiftKey && group.attrs.initialAspectRatio) {
			const contentInitialAspectRatio = group.attrs.initialAspectRatio;
			const anchorName = activeAnchor.name();

			let oppositeAnchor: Circle | undefined;
			if (anchorName === 'topLeft') oppositeAnchor = bottomRight;
			else if (anchorName === 'topRight') oppositeAnchor = bottomLeft;
			else if (anchorName === 'bottomRight') oppositeAnchor = topLeft;
			else if (anchorName === 'bottomLeft') oppositeAnchor = topRight;

			if (oppositeAnchor) {
			const ox = oppositeAnchor.x();
			const oy = oppositeAnchor.y();

			let dx = anchorX - ox;
			let dy = anchorY - oy;

			if (Math.abs(dx / contentInitialAspectRatio) > Math.abs(dy)) {
				const new_abs_dy = Math.abs(dx / contentInitialAspectRatio);
				anchorY = oy + Math.sign(dy || (anchorName.includes('bottom') ? 1: -1) ) * new_abs_dy;
			} else {
				const new_abs_dx = Math.abs(dy * contentInitialAspectRatio);
				anchorX = ox + Math.sign(dx || (anchorName.includes('Right') ? 1: -1) ) * new_abs_dx;
			}
			activeAnchor.x(anchorX);
			activeAnchor.y(anchorY);
			}
		}

		switch (activeAnchor.getName()) {
			case 'topLeft':
			topRight.y(anchorY);
			bottomLeft.x(anchorX);
			break;
			case 'topRight':
			topLeft.y(anchorY);
			bottomRight.x(anchorX);
			break;
			case 'bottomRight':
			bottomLeft.y(anchorY);
			topRight.x(anchorX);
			break;
			case 'bottomLeft':
			bottomRight.y(anchorY);
			topLeft.x(anchorX);
			break;
		}

		let finalContentPosX = topLeft.x();
		let finalContentPosY = topLeft.y();
		let finalWidth = topRight.x() - topLeft.x();
		let finalHeight = bottomLeft.y() - topLeft.y();

		if (finalWidth < 0) {
			finalContentPosX = topRight.x();
			finalWidth = -finalWidth;
		}
		if (finalHeight < 0) {
			finalContentPosY = bottomLeft.y();
			finalHeight = -finalHeight;
		}

		if (contentNode) {
			contentNode.position({ x: finalContentPosX, y: finalContentPosY });
			if (finalWidth >= 0 && finalHeight >= 0) {
			contentNode.width(finalWidth);
			contentNode.height(finalHeight);
			}
		}
		activeAnchor.getLayer()?.batchDraw();
	}

	function addAnchor(group: Group, x: number, y: number, name: string) {
		const anchor = new Konva.Circle({
			x: x,
			y: y,
			stroke: '#666',
			fill: '#ddd',
			strokeWidth: 2,
			radius: 8,
			name: name,
			draggable: true,
			dragOnTop: false,
		});

		anchor.on('dragmove', function (evt) {
			update(this, evt);
		});
		
		anchor.on('mousedown touchstart', function () {
			group.draggable(false);
			this.moveToTop();
		});
		
		anchor.on('dragend', function () {
			group.draggable(true);
			this.getLayer()?.batchDraw();
		});
		
		anchor.on('mouseover', function () {
			const container = this.getStage()?.container();
			if (container) container.style.cursor = 'pointer';
			this.strokeWidth(4);
			this.getLayer()?.batchDraw();
		});
		
		anchor.on('mouseout', function () {
			const container = this.getStage()?.container();
			if (container) container.style.cursor = 'default';
			this.strokeWidth(2);
			this.getLayer()?.batchDraw();
		});

		group.add(anchor);
	}

	function rebindAnchorEvents(anchor: Circle, group: Group) {
		anchor.off('dragmove mousedown touchstart dragend mouseover mouseout'); 

		anchor.on('dragmove', function (evt) {
			update(this, evt);
		});
		anchor.on('mousedown touchstart', function () {
			group.draggable(false);
			this.moveToTop();
		});
		anchor.on('dragend', function () {
			group.draggable(true);
			this.getLayer()?.batchDraw();
		});
		anchor.on('mouseover', function () {
			const currentStageContainer = this.getStage()?.container(); 
			if (currentStageContainer) currentStageContainer.style.cursor = 'pointer';
			this.strokeWidth(4);
			this.getLayer()?.batchDraw();
		});
		anchor.on('mouseout', function () {
			const currentStageContainer = this.getStage()?.container();
			if (currentStageContainer) currentStageContainer.style.cursor = 'default';
			this.strokeWidth(2);
			this.getLayer()?.batchDraw();
		});
	}

	function handleImageUpload(event: Event) {
		const currentActiveLayer = $activeLayer; // Get from store
		if (!currentActiveLayer || !stage) return;

		const target = event.target as HTMLInputElement;
		const file = target.files?.[0];

		if (file) { 
			const reader = new FileReader();
			reader.onload = function (e) {
				const img = new window.Image();
				img.onload = function () {
					const MAX_DIMENSION = 400;
					let newWidth, newHeight;

					if (img.width > img.height) {
						newWidth = MAX_DIMENSION;
						newHeight = (img.height / img.width) * MAX_DIMENSION;
					} else {
						newHeight = MAX_DIMENSION;
						newWidth = (img.width / img.height) * MAX_DIMENSION;
					}

					const imageGroup = new Konva.Group({
						x: (stage?.width() ?? 0) / 2 - newWidth / 2,
						y: (stage?.height() ?? 0) / 2 - newHeight / 2,
						draggable: true,
					});
					
					imageGroup.attrs.type = 'image';
					imageGroup.attrs.initialAspectRatio = newHeight > 0 ? newWidth / newHeight : 1;
					currentActiveLayer.add(imageGroup);

					const konvaImage = new Konva.Image({
						x: 0,
						y: 0,
						image: img, 
						width: newWidth,
						height: newHeight,
						name: 'Image',
					});
					imageGroup.add(konvaImage);

					addAnchor(imageGroup, 0, 0, 'topLeft');
					addAnchor(imageGroup, newWidth, 0, 'topRight');
					addAnchor(imageGroup, newWidth, newHeight, 'bottomRight');
					addAnchor(imageGroup, 0, newHeight, 'bottomLeft');
					currentActiveLayer.batchDraw();
				};
				img.src = e.target?.result as string; 
			};
			reader.readAsDataURL(file);
			target.value = ''; // Reset file input
		}
	}

	function addTextElement() {
		const currentActiveLayer = $activeLayer;
		if (!stage || !currentActiveLayer) return;

		textCounter++;
		const initialWidth = 200;
		const initialHeight = 50; 

		const textGroup = new Konva.Group({
			x: (stage.width() ?? 0) / 2 - initialWidth / 2,
			y: (stage.height() ?? 0) / 2 - initialHeight / 2,
			draggable: true,
			name: 'textGroup' + textCounter,
		});
		textGroup.attrs.type = 'text';
		textGroup.attrs.initialAspectRatio = initialWidth / initialHeight;
		textGroup.attrs.initialWidth = initialWidth;
		textGroup.attrs.initialHeight = initialHeight;
		currentActiveLayer.add(textGroup);

		const konvaText = new Konva.Text({
			x: 0,
			y: 0,
			text: 'Sample Text',
			fontSize: 20,
			fontFamily: selectedFontFamilyForNewText,
			fill: 'black',
			width: initialWidth,
			height: initialHeight,
			name: 'textContent',
			padding: 5,
			align: 'center',
			verticalAlign: 'middle',
		});
		konvaText.attrs.initialFontSize = 20;
		textGroup.add(konvaText);

		addAnchor(textGroup, 0, 0, 'topLeft');
		addAnchor(textGroup, initialWidth, 0, 'topRight');
		addAnchor(textGroup, initialWidth, initialHeight, 'bottomRight');
		addAnchor(textGroup, 0, initialHeight, 'bottomLeft');

		konvaText.on('dblclick dbltap', () => {
			const currentText = konvaText.text();
			const newText = prompt("Edit text:", currentText);
			if (newText !== null && newText !== currentText) {
				konvaText.text(newText);
				currentActiveLayer.batchDraw();
			}
		});
		currentActiveLayer.batchDraw();
	}

	async function handleFontUpload(event: Event) {
		const target = event.target as HTMLInputElement;
		const file = target.files?.[0];
		if (file) { // No direct dependency on layer or stage for font loading itself
			const reader = new FileReader();
			reader.onload = async function (e) {
				try {
					let fontFamilyName = file.name.split('.')[0].replace(/[^a-zA-Z0-9]/g, '');
					if (!fontFamilyName) fontFamilyName = 'CustomFont' + Date.now();

					let tempName = fontFamilyName;
					let counter = 1;
					while ($availableFonts.includes(tempName)) {
						tempName = `${fontFamilyName}${counter++}`;
					}
					fontFamilyName = tempName;

					const fontFace = new FontFace(fontFamilyName, `url(${e.target?.result as string})`);
					await fontFace.load();
					document.fonts.add(fontFace);
					availableFonts.update(fonts => [...fonts, fontFamilyName]);
					selectedFontFamilyForNewText = fontFamilyName;
					alert(`Font '${fontFamilyName}' loaded successfully!`);
				} catch (error) {
					console.error('Font loading failed:', error);
					alert('Failed to load font.');
				}
			};
			reader.readAsDataURL(file);
			target.value = '';
		}
	}

	function exportData() {
		if (!stage || $pagesStore.length === 0) {
			alert("Nothing to export.");
			return;
		}

		const pagesData = $pagesStore.map(pageLayer => {
			const images = pageLayer.find('Image');
			images.forEach(konvaImageNode => {
				const htmlImageElement = konvaImageNode.image() as HTMLImageElement | undefined;
				if (htmlImageElement && htmlImageElement.src && htmlImageElement.src.startsWith('data:image')) {
					konvaImageNode.setAttr('imageDataURL', htmlImageElement.src);
				}
			});
			return JSON.parse(pageLayer.toJSON()); // pageLayer.toJSON() returns a string
		});

		const exportObject = {
			version: "zinemaker-v2",
			stageConfig: {
				width: stage.width(),
				height: stage.height()
			},
			availableFonts: $availableFonts, // Exporting the list of font names
			pages: pagesData
		};

		const json = JSON.stringify(exportObject, null, 2);
		const blob = new Blob([json], { type: 'application/json' });
		const url = URL.createObjectURL(blob);
		const a = document.createElement('a');
		a.href = url;
		a.download = 'zinemaker-save.json';
		document.body.appendChild(a);
		a.click();
		document.body.removeChild(a);
		URL.revokeObjectURL(url);
	}

	async function handleImport(event: Event) {
		const target = event.target as HTMLInputElement;
		const file = target.files?.[0];
		const containerId = "container";

		if (file) {
			const reader = new FileReader();
			reader.onload = async function (e) {
				const jsonString = e.target?.result as string;
				let importObject: any;
				try {
					importObject = JSON.parse(jsonString);
				} catch (error) {
					alert("Failed to parse JSON file.");
					console.error("JSON Parse Error:", error);
					return;
				}

				if (!importObject.pages || !importObject.stageConfig || importObject.version !== "zinemaker-v2") {
					alert("Invalid or outdated ZineMaker JSON file format.");
					return;
				}

				if (stage) {
					stage.destroy();
					stage = null;
				}
				pagesStore.set([]);
				currentPageIndexStore.set(0);

				if (importObject.availableFonts) {
					// This sets the list of font names. Actual font data (files) are not in the JSON.
					// User would need to re-upload custom fonts if they are not globally available.
					availableFonts.set(importObject.availableFonts);
				}
				
				await tick(); // Ensure old stage is gone, DOM is ready for new one

				const containerElement = document.getElementById(containerId);
				const containerWidth = containerElement ? containerElement.offsetWidth : importObject.stageConfig.width || 595;
				const containerHeight = importObject.stageConfig.height || (window.innerHeight * 0.8);

				stage = new Konva.Stage({
					container: containerId,
					width: containerWidth,
					height: containerHeight,
				});

				const newPages: Layer[] = [];
				const allImageLoadPromises: Promise<void>[] = [];

				for (const pageData of importObject.pages) {
					// pageData is the parsed object from layer.toJSON()
					const newLayer = Konva.Node.create(pageData, undefined) as Layer; // Create layer from its JSON representation

					const konvaImages = newLayer.find('Image');
					konvaImages.forEach(konvaImg => {
						const imageDataURL = konvaImg.getAttr('imageDataURL');
						if (imageDataURL) {
						const promise = new Promise<void>((resolve, reject) => {
							const htmlImg = new window.Image();
							htmlImg.onload = () => {
								konvaImg.image(htmlImg); 
								resolve();
							};
							htmlImg.onerror = (err) => {
								console.error('Failed to load image from data URL during import:', err);
								reject(err);
							};
							htmlImg.src = imageDataURL;
						});
						allImageLoadPromises.push(promise);
						}
					});

					const groups = newLayer.getChildren(node => node.getClassName() === 'Group') as Group[];
					groups.forEach(group => {
						if (typeof group.attrs.initialAspectRatio !== 'number') {
							const imageNode = group.findOne(n => n.getClassName() === 'Image');
							const textContentNode = group.findOne(n => n.getClassName() === 'Text' && n.name() === 'textContent');
							let contentToMeasure = imageNode || textContentNode;
							if (contentToMeasure && contentToMeasure.height() !== undefined && contentToMeasure.height() !== 0) {
								group.attrs.initialAspectRatio = contentToMeasure.width() / contentToMeasure.height();
							} else {
								group.attrs.initialAspectRatio = 1; 
							}
						}

						const anchors = group.getChildren(node => node.getClassName() === 'Circle' && ['topLeft', 'topRight', 'bottomRight', 'bottomLeft'].includes(node.name())) as Circle[];
						anchors.forEach(anchor => {
							rebindAnchorEvents(anchor, group);
						});

						if (group.attrs.type === 'text') {
							const textNode = group.findOne(n => n.getClassName() === 'Text' && n.name() === 'textContent') as Konva.Text | undefined;
							if (textNode) {
								textNode.off('dblclick dbltap'); 
								textNode.on('dblclick dbltap', () => {
									const currentText = textNode.text();
									const newText = prompt("Edit text:", currentText);
									if (newText !== null && newText !== currentText) {
										textNode.text(newText);
										newLayer.batchDraw();
									}
								});
							}
						}
					});
					newPages.push(newLayer);
				}

				try {
					await Promise.all(allImageLoadPromises);
					console.log('All images from JSON reloaded.');
				} catch (err) {
					console.error('Error reloading one or more images from JSON:', err);
				}
				
				pagesStore.set(newPages);
				currentPageIndexStore.set(0); // Triggers reactive update to add first layer
				
				// Ensure stage is drawn after all setup
				await tick();
				stage?.batchDraw();

				console.log('Import successful.');
			};
			reader.readAsText(file);
			target.value = ''; 
		}
	}
</script>

<!-- Pagination Controls -->
<div style="margin-bottom: 10px; padding: 5px; border: 1px solid #eee;">
	<button on:click={prevPage} disabled={$currentPageIndexStore === 0}>Previous Page</button>
	<span style="margin: 0 10px;">
		Page {$currentPageIndexStore + 1} of {$pagesStore.length}
	</span>
	<button on:click={nextPage} disabled={$currentPageIndexStore >= $pagesStore.length - 1 && $pagesStore.length > 0}>Next Page</button>
	<button on:click={addBlankPage} style="margin-left: 10px;">Add New Page</button>
</div>

<!-- Existing Controls -->
<input type="file" id="imageUploader" accept="image/*" on:change={handleImageUpload} />
<button on:click={exportData}>Export JSON</button>
<input type="file" id="jsonUploader" accept=".json" on:change={handleImport} style="margin-left: 10px;" />
<button on:click={addTextElement} style="margin-left: 10px;">Add Text</button>
<input type="file" id="fontUploader" accept=".ttf,.otf,.woff,.woff2" on:change={handleFontUpload} style="margin-left: 10px;" />
<label for="fontSelector" style="margin-left: 10px;">Font for new text:</label>
<select id="fontSelector" bind:value={selectedFontFamilyForNewText}>
	{#each $availableFonts as font}
		<option value={font}>{font}</option>
	{/each}
</select>

<div id="container"></div>

<style>
  #container {
	border: 5px solid #ccc;
	width: 50%; /* Or your preferred width */
	min-height: 600px; /* Ensure it has some height */
	box-sizing: border-box;
	background-color: #f9f9f9; /* Light background for visibility */
  }
  button, input, select, label {
	margin-bottom: 5px;
	margin-right: 5px; /* Add some spacing */
  }
</style>