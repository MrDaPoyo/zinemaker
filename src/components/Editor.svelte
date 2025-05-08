<div id="container"></div>
<script>	
import Konva from "konva";
import { onMount } from 'svelte';

let stage;

onMount(() => {
	stage = new Konva.Stage({
		container: "container",
		width: window.innerWidth,
		height: window.innerHeight,
	});

	const layer = new Konva.Layer();
	stage.add(layer);

	const circle = new Konva.Circle({
		x: stage.width() / 2,
		y: stage.height() / 2,
		radius: 70,
		fill: "red",
		stroke: "black",
		strokeWidth: 4,
		draggable: true,
	});

	// add cursor styling
	circle.on("mouseover", function () {
		document.body.style.cursor = "pointer";
	});
	layer.add(circle);
});
</script>