<input type="file" id="imageUploader" accept="image/*" on:change={handleImageUpload} />
<div id="container"></div>

<script>
import Konva from "konva";
import { onMount, onDestroy } from 'svelte';

let stage;
let layer;

onMount(() => {
	stage = new Konva.Stage({
		container: "container",
		width: window.innerWidth,
		height: window.innerHeight,
	});

	layer = new Konva.Layer();
	stage.add(layer);

	const handleResize = () => {
		if (stage) {
			stage.width(window.innerWidth);
			stage.height(window.innerHeight);
		}
	};

	window.addEventListener('resize', handleResize);

	return () => {
		window.removeEventListener('resize', handleResize);
		if (stage) {
			stage.destroy();
		}
	};
});

function update(activeAnchor, evt) {
  const group = activeAnchor.getParent();
  const topLeft = group.findOne('.topLeft');
  const topRight = group.findOne('.topRight');
  const bottomRight = group.findOne('.bottomRight');
  const bottomLeft = group.findOne('.bottomLeft');
  const image = group.findOne('Image');

  let anchorX = activeAnchor.x();
  let anchorY = activeAnchor.y();

  if (evt && evt.evt.shiftKey && group.attrs.initialAspectRatio) {
	const imageInitialAspectRatio = group.attrs.initialAspectRatio;
	const anchorName = activeAnchor.getName();

	let oppositeAnchor;
	if (anchorName === 'topLeft') oppositeAnchor = bottomRight;
	else if (anchorName === 'topRight') oppositeAnchor = bottomLeft;
	else if (anchorName === 'bottomRight') oppositeAnchor = topLeft;
	else if (anchorName === 'bottomLeft') oppositeAnchor = topRight;

	if (oppositeAnchor) {
	  const ox = oppositeAnchor.x();
	  const oy = oppositeAnchor.y();

	  let dx = anchorX - ox;
	  let dy = anchorY - oy;

	  if (Math.abs(dx / imageInitialAspectRatio) > Math.abs(dy)) {
		const new_abs_dy = Math.abs(dx / imageInitialAspectRatio);
		anchorY = oy + Math.sign(dy || (anchorName.includes('bottom') ? 1: -1) ) * new_abs_dy; // ensure sign if dy is 0
	  } else {
		const new_abs_dx = Math.abs(dy * imageInitialAspectRatio);
		anchorX = ox + Math.sign(dx || (anchorName.includes('Right') ? 1: -1) ) * new_abs_dx; // ensure sign if dx is 0
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

  let finalImageX = topLeft.x();
  let finalImageY = topLeft.y();
  let finalWidth = topRight.x() - topLeft.x();
  let finalHeight = bottomLeft.y() - topLeft.y();

  if (finalWidth < 0) {
	finalImageX = topRight.x();
	finalWidth = -finalWidth;
  }
  if (finalHeight < 0) {
	finalImageY = bottomLeft.y();
	finalHeight = -finalHeight;
  }

  image.position({ x: finalImageX, y: finalImageY });

  if (finalWidth >= 0 && finalHeight >= 0) {
	image.width(finalWidth);
	image.height(finalHeight);
  }
}

function addAnchor(group, x, y, name) {
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
  });
  
  anchor.on('mouseover', function () {
	const container = stage.container();
	if (container) container.style.cursor = 'pointer';
	this.strokeWidth(4);
  });
  
  anchor.on('mouseout', function () {
	const container = stage.container();
	if (container) container.style.cursor = 'default';
	this.strokeWidth(2);
  });

  group.add(anchor);
}

function handleImageUpload(event) {
	const file = event.target.files[0];
	if (file && layer && stage) {
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
					x: stage.width() / 2 - newWidth / 2,
					y: stage.height() / 2 - newHeight / 2,
					draggable: true,
				});
				if (newHeight > 0) { // store the aspect ratio only if the height is not a zero
					imageGroup.attrs.initialAspectRatio = newWidth / newHeight;
				} else {
					imageGroup.attrs.initialAspectRatio = 1; // use the default aspect ratio, square
				}
				layer.add(imageGroup);

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
			};
			img.src = e.target.result;
		};
		reader.readAsDataURL(file);
	}
}
</script>

<style>
  #container {
	border: 1px solid #ccc;
  }

  #imageUploader {
	display: block;
	margin: 10px;
  }

  :global(body, html) {
	margin: 0;
	padding: 0;
  }
</style>