<script lang="ts">
	let imageCanvas: HTMLCanvasElement;
	let roomCanvas: HTMLCanvasElement;
	let files: FileList;

	// factor the image is scaled by on the canvas
	// load room picture
	let scaleFactor = 1;
	$: if (files && files[0]) {
		const reader = new FileReader();
		const ctx = imageCanvas.getContext('2d');
		reader.onload = (evt) => {
			const image = new Image();
			console.log('loaded reader');
			image.onload = () => {
				for (const canvas of [roomCanvas, imageCanvas]) {
					canvas.width = image.width;
					canvas.height = image.height;
					scaleFactor = 1000 / image.width;

					canvas.style.transform = `scale(${scaleFactor})`;
				}
				ctx!.drawImage(image, 0, 0);
			};
			image.src = (evt.target?.result as string) ?? '';
		};
		reader.readAsDataURL(files[0]);
	}

	interface Room {
		name: string;
		x: number;
		y: number;
		height: number;
		width: number;
		factor: number;
	}

	let totalSize: number;
	let finishedRooms: Room[] = [];
	let currentRoom: Room | undefined;
	$: allRooms = currentRoom ? [...finishedRooms, currentRoom] : [...finishedRooms];
	$: pixelPerUnit = calculatePixelsPerUnit(allRooms, totalSize);
	$: drawRooms(allRooms, pixelPerUnit);

	function handleMousedown(event: MouseEvent) {
		const { left, top } = roomCanvas.getBoundingClientRect();
		currentRoom = {
			name: `Room ${finishedRooms.length + 1}`,
			x: (event.clientX - left) / scaleFactor,
			y: (event.clientY - top) / scaleFactor,
			height: 0,
			width: 0,
			factor: 1
		};
	}

	function handleMouseup() {
		if (currentRoom) {
			if (currentRoom.x > 1 && currentRoom.y > 1) {
				finishedRooms = [...finishedRooms, currentRoom];
			}
			currentRoom = undefined;
		}
	}

	function handleMouseMove(event: MouseEvent) {
		if (currentRoom) {
			const { left, top } = roomCanvas.getBoundingClientRect();
			currentRoom.width = (event.clientX - left) / scaleFactor - currentRoom.x;
			currentRoom.height = (event.clientY - top) / scaleFactor - currentRoom.y;
		}
	}

	function calculatePixelsPerUnit(allRooms: Room[], totalSize: number) {
		return Math.sqrt(
			totalSize /
				allRooms.reduce((prev, room) => prev + Math.abs(room.height * room.width * room.factor), 0)
		);
	}

	function drawRooms(allRooms: Room[], pixelPerUnit: number) {
		if (!roomCanvas) return;
		const ctx = roomCanvas.getContext('2d');
		if (!ctx) return;

		ctx.clearRect(0, 0, roomCanvas.width, roomCanvas.height);
		ctx.textAlign = 'center';
		ctx.font = '40px';
		ctx.lineWidth = 10;

		for (const room of allRooms) {
			const centerX = room.x + room.width / 2;
			const centerY = room.y + room.height / 2;
			ctx.globalAlpha = 0.2;
			ctx.fillStyle = 'purple';

			ctx.fillRect(room.x, room.y, room.width, room.height);
			ctx.globalAlpha = 1;
			ctx.fillText(
				`${room.name}\n${printScaled(room.width * room.height * pixelPerUnit ** 2)}`,
				centerX,
				centerY
			);
			ctx.fillText(printScaled(room.width * pixelPerUnit), centerX, room.y);
			ctx.fillText(printScaled(room.height * pixelPerUnit), room.x, centerY);
		}
	}

	function printScaled(x: number): string {
		return Math.abs(x).toFixed(2);
	}

	function resetRooms() {
		finishedRooms = [];
		currentRoom = undefined;
	}
</script>

<div class="columns">
	<div
		class="has-background-primary is-one-third column  m-3"
		style="display: flex; flex-direction: column; justify-content: center; align-items: center"
	>
		<span class="title">1. Upload floorplan</span>
		<div class="file">
			<label class="file-label field">
				<span class="file-cta">
					<span class="file-icon">
						<ion-icon name="document-outline" />
					</span>
					<span class="file-label"> Choose an image </span>
				</span>
				<input class="file-input" type="file" name="resume" bind:files on:change={resetRooms} />
			</label>
		</div>

		<span class="title">2. Enter total area</span>
		<div class="field">
			<p class="control has-icons-left">
				<span class="icon is-small is-left">
					<ion-icon name="move-outline" />
				</span>
				<input
					type="text"
					class="input is-primary"
					placeholder="Total areaa (in sqft or m^2)"
					bind:value={totalSize}
				/>
			</p>
		</div>

		<span class="title">3. Draw rooms</span>
		<table class="table" style="width: 100%;">
			<thead>
				<tr class="thead">
					<th>Room</th>
					<th>Width</th>
					<th>Height</th>
					<th>Area</th>
					<th>Multiplier</th>
				</tr>
			</thead>
			<tbody>
				{#each finishedRooms as room, i (i)}
					<tr class="tbody">
						<td>
							<div class="field">
								<p class="control">
									<input type="text" class="input" style="width: 10vw;" bind:value={room.name} />
								</p>
							</div>
						</td>
						<td>
							{printScaled(room.height * pixelPerUnit)}
						</td>
						<td>
							{printScaled(room.width * pixelPerUnit)}
						</td>
						<td>
							<!-- * pixelsPerUnit for square -->
							{printScaled(room.height * room.width * pixelPerUnit ** 2)}
						</td>
						<td style="width: min-contentq;">
							<input type="number" class="input" style="width: 5vw;" bind:value={room.factor} />
							<button
								class="button"
								on:click={() => (finishedRooms = finishedRooms.filter((r) => r != room))}
							>
								<span class="icon">
									<ion-icon name="trash-outline" />
								</span>
							</button>
						</td>
					</tr>
				{:else}
					<span class="title is-4" style="color:black;">Draw on the canvas to add rooms</span>
				{/each}
			</tbody>
		</table>
	</div>

	<div class="wrapper is-two-thirds column">
		<canvas
			bind:this={roomCanvas}
			class="roomCanvas"
			on:mousemove={handleMouseMove}
			on:mousedown={handleMousedown}
			on:mouseup={handleMouseup}
		/>
		<canvas bind:this={imageCanvas} />
	</div>
</div>

<style>
	.wrapper {
		position: relative;
		height: 100vh;
	}
	.wrapper canvas {
		transform-origin: top left;
		position: absolute;
		top: 0;
		left: 0;
	}

	.roomCanvas {
		z-index: 1;
	}

	.columns {
		font-family: BlinkMacSystemFont, -apple-system, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell,
			'Fira Sans', 'Droid Sans', 'Helvetica Neue', Helvetica, Arial, sans-serif;
	}

	.title {
		color: #ffffff;
	}
</style>
