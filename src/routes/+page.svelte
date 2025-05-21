<script>
	// Import the onMount function from Svelte to run code when the component loads
	import { onMount } from "svelte";

	// Variables to store references to the board element and chess game
	let board;
	let chess;

	// The starting positions of all chess pieces on the board
	// Each letter represents a piece (lowercase = black, uppercase = white)
	// r=rook, n=knight, b=bishop, q=queen, k=king, p=pawn
	const initialPositions = [
		["r", "n", "b", "q", "k", "b", "n", "r"], // Black back row
		["p", "p", "p", "p", "p", "p", "p", "p"], // Black pawns
		["", "", "", "", "", "", "", ""], // Empty rows
		["", "", "", "", "", "", "", ""],
		["", "", "", "", "", "", "", ""],
		["", "", "", "", "", "", "", ""],
		["P", "P", "P", "P", "P", "P", "P", "P"], // White pawns
		["R", "N", "B", "Q", "K", "B", "N", "R"], // White back row
	];

	// URLs for images of each chess piece (using Wikimedia Commons images)
	const pieceImages = {
		r: "https://upload.wikimedia.org/wikipedia/commons/a/a0/Chess_rdt60.png", // black rook
		n: "https://upload.wikimedia.org/wikipedia/commons/f/f1/Chess_ndt60.png", // black knight
		b: "https://upload.wikimedia.org/wikipedia/commons/8/81/Chess_bdt60.png", // black bishop
		q: "https://upload.wikimedia.org/wikipedia/commons/a/af/Chess_qdt60.png", // black queen
		k: "https://upload.wikimedia.org/wikipedia/commons/e/e3/Chess_kdt60.png", // black king
		p: "https://upload.wikimedia.org/wikipedia/commons/c/cd/Chess_pdt60.png", // black pawn
		R: "https://upload.wikimedia.org/wikipedia/commons/5/5c/Chess_rlt60.png", // white rook
		N: "https://upload.wikimedia.org/wikipedia/commons/2/28/Chess_nlt60.png", // white knight
		B: "https://upload.wikimedia.org/wikipedia/commons/9/9b/Chess_blt60.png", // white bishop
		Q: "https://upload.wikimedia.org/wikipedia/commons/4/49/Chess_qlt60.png", // white queen
		K: "https://upload.wikimedia.org/wikipedia/commons/3/3b/Chess_klt60.png", // white king
		P: "https://upload.wikimedia.org/wikipedia/commons/0/04/Chess_plt60.png", // white pawn
	};

	// The main Chess class that handles all game logic
	class Chess {
		constructor(boardEl) {
			// Store reference to the board HTML element
			this.boardEl = boardEl;

			// Create a copy of the initial positions to track current board state
			this.positions = initialPositions.map((r) => [...r]);

			// Track castling rights (whether kings/rooks have moved)
			this.kingMoved = { w: false, b: false }; // Track if kings have moved
			this.rookMoved = {
				w: { a: false, h: false }, // White's a-file and h-file rooks
				b: { a: false, h: false }, // Black's a-file and h-file rooks
			};

			// Track currently dragged piece
			this.currentDrag = null;

			// Store the size of the board (width/height in pixels)
			this.size = 0;

			// Track en passant target square (if any)
			this.enPassantTarget = null;

			// Current turn (w = white, b = black)
			this.turn = "w";

			// Game over state
			this.gameOver = false;
			this.gameOverType = null; // 'checkmate' or 'stalemate'
			this.gameOverColor = null; // Which player is affected
		}

		// Initialize the chess board
		init() {
			this.generateBoard();
		}

		// Generate the visual chess board
		generateBoard() {
			// Clear any existing board
			this.clearBoard();

			// Set up the board container with CSS grid
			this.boardEl.className =
				"relative w-[30rem] aspect-square grid grid-cols-8 grid-rows-8";

			// Get the actual size of the board element
			this.size = this.boardEl.getBoundingClientRect().width;

			// Create all 64 squares (8x8 grid)
			for (let r = 0; r < 8; r++) {
				for (let f = 0; f < 8; f++) {
					// Create a square element
					const sq =
						document.createElement("div");

					// Store position data
					sq.dataset.rank = r;
					sq.dataset.file = f;

					// Alternate square colors (light/dark)
					const light = (r + f) % 2 === 0;
					sq.className = `relative w-full h-full ${
						light
							? "bg-[#f0d9b5]"
							: "bg-[#b58863]"
					}`;

					// Add the square to the board
					this.boardEl.appendChild(sq);
				}
			}

			// Place all the pieces on the board
			for (let r = 0; r < 8; r++) {
				for (let f = 0; f < 8; f++) {
					const p = this.positions[r][f];
					if (!p) continue; // Skip empty squares

					// Create an image element for the piece
					const img =
						document.createElement("img");
					img.src = pieceImages[p];
					img.alt = p;

					// Calculate positioning and size
					const cell = this.size / 8; // Size of one square
					const pSize = cell * 0.8; // Piece is 80% of square size
					const off = (cell - pSize) / 2; // Center the piece

					// Style the piece image
					img.style.position = "absolute";
					img.style.width = `${pSize}px`;
					img.style.height = `${pSize}px`;
					img.style.left = `${f * cell + off}px`;
					img.style.top = `${r * cell + off}px`;
					img.style.cursor = "pointer";
					img.style.zIndex = 1;

					// Add click/drag handlers
					img.addEventListener("mousedown", (e) =>
						this._onMouseDown(e, {
							r,
							f,
							type: p,
						}),
					);

					// Add the piece to the board
					this.boardEl.appendChild(img);
				}
			}

			// Highlight special states (check, checkmate, stalemate)
			if (this.gameOver) {
				// Game is over - highlight the king
				const { r, f } = this.gameOverColor;
				const sq = this.boardEl.querySelector(
					`div[data-rank="${r}"][data-file="${f}"]`,
				);
				sq.style.backgroundColor =
					this.gameOverType === "checkmate"
						? "#ff4d4d" // Red for checkmate
						: "#a0a0a0"; // Gray for stalemate
			} else if (this.isInCheck(this.turn)) {
				// Current player is in check - highlight their king
				const { r, f } = this.findKing(this.turn);
				const sq = this.boardEl.querySelector(
					`div[data-rank="${r}"][data-file="${f}"]`,
				);
				sq.style.backgroundColor = "#ff4d4d"; // Red
			}

			// Add mouse movement handlers for dragging pieces
			document.addEventListener(
				"mousemove",
				this._onMouseMove,
			);
			document.addEventListener("mouseup", this._onMouseUp);
		}

		// Clear the board and remove event listeners
		clearBoard() {
			document.removeEventListener(
				"mousemove",
				this._onMouseMove,
			);
			document.removeEventListener(
				"mouseup",
				this._onMouseUp,
			);
			this.boardEl.innerHTML = "";
		}

		// Reset the game to the starting position
		restart() {
			// Reset board positions
			this.positions = initialPositions.map((r) => [...r]);

			// Reset game state
			this.enPassantTarget = null;
			this.turn = "w";
			this.gameOver = false;
			this.gameOverType = null;
			this.gameOverColor = null;

			// Reset castling rights
			this.kingMoved = { w: false, b: false };
			this.rookMoved = {
				w: { a: false, h: false },
				b: { a: false, h: false },
			};

			// Redraw the board
			this.generateBoard();
		}

		// Find the position of a king of the given color
		findKing(color) {
			for (let r = 0; r < 8; r++)
				for (let f = 0; f < 8; f++) {
					const p = this.positions[r][f];
					if (
						p &&
						(color === "w"
							? p === "K"
							: p === "k")
					)
						return { r, f };
				}
			return { r: 0, f: 0 }; // Default return (shouldn't happen)
		}

		// Check if the given color is in check
		isInCheck(color) {
			// Find the king's position
			const { r: kr, f: kf } = this.findKing(color);
			const enemy = color === "w" ? "b" : "w"; // Opposite color

			// Check all enemy pieces to see if any can attack the king
			for (let r = 0; r < 8; r++) {
				for (let f = 0; f < 8; f++) {
					const p = this.positions[r][f];
					if (!p) continue; // Skip empty squares

					// Determine piece color
					const c =
						p === p.toUpperCase()
							? "w"
							: "b";
					if (c !== enemy) continue; // Skip friendly pieces

					// Get all legal moves for this piece
					const moves = this.getLegal(
						{ r, f },
						p,
						true, // Ignore check (just checking attack patterns)
					);

					// If any move targets the king's square, it's check
					if (
						moves.some(
							(m) =>
								m.r === kr &&
								m.f === kf,
						)
					) {
						return true;
					}
				}
			}
			return false;
		}

		// Helper to check if a square is attacked by any enemy pieces
		isAttacked(r, f, attackerColor) {
			for (let i = 0; i < 8; i++) {
				for (let j = 0; j < 8; j++) {
					const p = this.positions[i][j];
					if (!p) continue;

					// Determine piece color
					const c =
						p === p.toUpperCase()
							? "w"
							: "b";
					if (c !== attackerColor) continue;

					// Check if this piece can attack the target square
					if (
						this.getLegal(
							{ r: i, f: j },
							p,
							true,
						).some(
							(m) =>
								m.r === r &&
								m.f === f,
						)
					) {
						return true;
					}
				}
			}
			return false;
		}

		// Get all legal moves for a piece at a given position
		getLegal(from, type, ignoreCheck = false) {
			const { r, f } = from; // Current row and file (column)
			const b = this.positions; // Reference to the board

			// Helper functions:
			const inB = (nr, nf) =>
				nr >= 0 && nr < 8 && nf >= 0 && nf < 8; // Check if position is on board
			const isEmp = (nr, nf) => b[nr][nf] === ""; // Check if square is empty
			const isEnemy = (
				nr,
				nf, // Check if square has enemy piece
			) =>
				b[nr][nf] !== "" &&
				(b[nr][nf] === b[nr][nf].toUpperCase()) !==
					(type === type.toUpperCase());

			const lower = type.toLowerCase(); // Piece type in lowercase
			let moves = []; // Array to store all legal moves

			// ─── PAWN MOVEMENT ────────────────────────────────────────────
			if (lower === "p") {
				const dir = type === "P" ? -1 : 1; // White pawns move up (-1), black move down (1)
				const sr = type === "P" ? 6 : 1; // Starting row for this pawn

				// Forward move (1 square)
				const one = r + dir;
				if (inB(one, f) && isEmp(one, f))
					moves.push({ r: one, f });

				// Initial two-square move
				const two = r + dir * 2;
				if (
					r === sr && // On starting row
					inB(two, f) && // Two squares forward is on board
					isEmp(one, f) && // First square is empty
					isEmp(two, f) // Second square is empty
				)
					moves.push({ r: two, f });

				// Diagonal captures
				[-1, 1].forEach((df) => {
					const nr = r + dir;
					const nf = f + df;
					if (
						inB(nr, nf) &&
						(isEnemy(nr, nf) || // Normal capture
							(this.enPassantTarget && // Or en passant
								this
									.enPassantTarget
									.r ===
									nr &&
								this
									.enPassantTarget
									.f ===
									nf))
					)
						moves.push({ r: nr, f: nf });
				});
			}
			// ─── KNIGHT MOVEMENT ──────────────────────────────────────────
			else if (lower === "n") {
				// All 8 possible L-shaped moves
				[
					[2, 1],
					[2, -1],
					[-2, 1],
					[-2, -1],
					[1, 2],
					[1, -2],
					[-1, 2],
					[-1, -2],
				].forEach((d) => {
					const nr = r + d[0],
						nf = f + d[1];
					if (
						inB(nr, nf) &&
						(isEmp(nr, nf) ||
							isEnemy(nr, nf))
					)
						moves.push({ r: nr, f: nf });
				});
			}
			// ─── KING MOVEMENT ───────────────────────────────────────────
			else if (lower === "k") {
				// One-square king moves in all 8 directions
				for (let dr = -1; dr <= 1; dr++) {
					for (let df = -1; df <= 1; df++) {
						if (dr || df) {
							// Skip 0,0 (current square)
							const nr = r + dr,
								nf = f + df;
							if (
								inB(nr, nf) &&
								(isEmp(
									nr,
									nf,
								) ||
									isEnemy(
										nr,
										nf,
									))
							) {
								moves.push({
									r: nr,
									f: nf,
								});
							}
						}
					}
				}

				// ─── CASTLING MOVES ──────────────────────────────────────
				if (!ignoreCheck) {
					const color = type === "K" ? "w" : "b";
					const enemy = color === "w" ? "b" : "w";

					// Can only castle if king hasn't moved and isn't in check
					if (
						!this.kingMoved[color] &&
						!this.isInCheck(color)
					) {
						// Kingside castling (short castle)
						if (
							!this.rookMoved[color]
								.h && // Kingside rook hasn't moved
							inB(r, f + 2) && // Target square is on board
							isEmp(r, f + 1) && // Squares between are empty
							isEmp(r, f + 2) &&
							!this.isAttacked(
								r,
								f + 1,
								enemy,
							) && // Squares aren't attacked
							!this.isAttacked(
								r,
								f + 2,
								enemy,
							)
						) {
							moves.push({
								r,
								f: f + 2,
							});
						}

						// Queenside castling (long castle)
						if (
							!this.rookMoved[color]
								.a && // Queenside rook hasn't moved
							inB(r, f - 3) && // Target square is on board
							isEmp(r, f - 1) && // Squares between are empty
							isEmp(r, f - 2) &&
							isEmp(r, f - 3) &&
							!this.isAttacked(
								r,
								f - 1,
								enemy,
							) && // Squares aren't attacked
							!this.isAttacked(
								r,
								f - 2,
								enemy,
							)
						) {
							moves.push({
								r,
								f: f - 2,
							});
						}
					}
				}
			}
			// ─── BISHOP/QUEEN/ROOK MOVEMENT ──────────────────────────────
			else {
				const dirs = [];
				// Bishops and queens move diagonally
				if (lower === "b" || lower === "q")
					dirs.push(
						[1, 1],
						[1, -1],
						[-1, 1],
						[-1, -1],
					);
				// Rooks and queens move straight
				if (lower === "r" || lower === "q")
					dirs.push(
						[1, 0],
						[-1, 0],
						[0, 1],
						[0, -1],
					);

				// For each possible direction
				dirs.forEach((d) => {
					let nr = r + d[0],
						nf = f + d[1];
					// Keep moving in that direction until edge of board
					while (inB(nr, nf)) {
						if (isEmp(nr, nf)) {
							// Empty square - can move here
							moves.push({
								r: nr,
								f: nf,
							});
						} else {
							// Occupied square - can capture if enemy
							if (isEnemy(nr, nf))
								moves.push({
									r: nr,
									f: nf,
								});
							break; // Can't move past any piece
						}
						nr += d[0]; // Continue in direction
						nf += d[1];
					}
				});
			}

			// Filter out moves that would leave king in check
			if (!ignoreCheck) {
				moves = moves.filter((to) => {
					// Create a copy of the board to test the move
					const copy = this.positions.map(
						(row) => [...row],
					);
					copy[to.r][to.f] = copy[r][f]; // Move piece
					copy[r][f] = ""; // Clear original square

					// Temporarily swap to the copied board
					const tmp = this.positions;
					this.positions = copy;

					// Check if this move leaves king in check
					const inChk = this.isInCheck(
						type === type.toUpperCase()
							? "w"
							: "b",
					);

					// Restore the original board
					this.positions = tmp;
					return !inChk; // Keep move if it doesn't leave king in check
				});
			}

			return moves;
		}

		// Move a piece from one position to another
		movePiece(from, to) {
			const t = this.positions[from.r][from.f]; // The moving piece
			const color = t === t.toUpperCase() ? "w" : "b"; // Piece color

			// ─── EN PASSANT CAPTURE ──────────────────────────────────────
			if (
				t.toLowerCase() === "p" && // Moving piece is pawn
				this.enPassantTarget && // En passant target exists
				this.enPassantTarget.r === to.r && // Moving to en passant square
				this.enPassantTarget.f === to.f
			) {
				// Remove the captured pawn (which is diagonally behind)
				this.positions[from.r][to.f] = "";
			}

			// Reset en passant target (only valid for one move)
			this.enPassantTarget = null;

			// Set new en passant target if pawn moves two squares
			if (
				t.toLowerCase() === "p" &&
				Math.abs(to.r - from.r) === 2 // Two-square move
			) {
				this.enPassantTarget = {
					r: (from.r + to.r) / 2, // Square behind the pawn
					f: from.f,
				};
			}

			// Move the piece to the new square
			this.positions[to.r][to.f] = t;
			this.positions[from.r][from.f] = "";

			// ─── HANDLE CASTLING ROOK MOVE ───────────────────────────────
			if (
				t.toLowerCase() === "k" && // Moving piece is king
				Math.abs(to.f - from.f) === 2 // Moving two squares (castling)
			) {
				const homeRow = color === "w" ? 7 : 0; // Back row for this color
				const rookChar = color === "w" ? "R" : "r"; // Rook character

				if (to.f > from.f) {
					// Kingside castling (h-file rook moves to f-file)
					this.positions[homeRow][5] = rookChar;
					this.positions[homeRow][7] = "";
					this.rookMoved[color].h = true; // Mark rook as moved
				} else {
					// Queenside castling (a-file rook moves to d-file)
					this.positions[homeRow][3] = rookChar;
					this.positions[homeRow][0] = "";
					this.rookMoved[color].a = true; // Mark rook as moved
				}
			}

			// ─── UPDATE MOVED FLAGS FOR CASTLING RIGHTS ──────────────────
			if (t === "K")
				this.kingMoved.w = true; // White king moved
			else if (t === "k")
				this.kingMoved.b = true; // Black king moved
			else if (t === "R") {
				// White rook moved
				if (from.r === 7 && from.f === 0)
					// a1 rook
					this.rookMoved.w.a = true;
				if (from.r === 7 && from.f === 7)
					// h1 rook
					this.rookMoved.w.h = true;
			} else if (t === "r") {
				// Black rook moved
				if (from.r === 0 && from.f === 0)
					// a8 rook
					this.rookMoved.b.a = true;
				if (from.r === 0 && from.f === 7)
					// h8 rook
					this.rookMoved.b.h = true;
			}

			// ─── PAWN PROMOTION ─────────────────────────────────────────
			if (t === "P" && to.r === 0)
				// White pawn reaches 8th rank
				this.positions[to.r][to.f] = "Q"; // Promote to queen
			if (t === "p" && to.r === 7)
				// Black pawn reaches 1st rank
				this.positions[to.r][to.f] = "q"; // Promote to queen

			// Switch turns
			this.turn = this.turn === "w" ? "b" : "w";

			// Check for game over conditions (checkmate or stalemate)
			const inChk = this.isInCheck(this.turn); // Is new player in check?
			const anyMoves = this.positions
				.flatMap((row, r0) =>
					row.map((p, f0) => ({ p, r0, f0 })),
				)
				.filter(
					(x) =>
						x.p &&
						(x.p === x.p.toUpperCase()
							? "w"
							: "b") === this.turn,
				)
				.some(
					({ p, r0, f0 }) =>
						this.getLegal(
							{ r: r0, f: f0 },
							p,
						).length > 0,
				);

			// Determine game outcome
			if (inChk && !anyMoves) {
				// Checkmate - player is in check with no legal moves
				this.gameOver = true;
				this.gameOverType = "checkmate";
				this.gameOverColor = this.findKing(this.turn);
			} else if (!inChk && !anyMoves) {
				// Stalemate - player has no legal moves but isn't in check
				this.gameOver = true;
				this.gameOverType = "stalemate";
				this.gameOverColor = this.findKing(this.turn);
			}

			// Redraw the board with the new position
			this.generateBoard();
		}

		// Handle mouse down on a piece (start of drag)
		_onMouseDown = (e, { r, f, type }) => {
			if (this.gameOver) return; // Ignore if game is over

			// Only allow moving pieces of the current turn's color
			if (
				(type === type.toUpperCase() ? "w" : "b") !==
				this.turn
			)
				return;

			e.preventDefault();

			// Get all legal moves for this piece
			const legal = this.getLegal({ r, f }, type);
			if (!legal.length) return; // No moves available

			// Show available moves as dots on the board
			legal.forEach((m) => {
				const dot = document.createElement("div");
				const isCapture =
					this.positions[m.r][m.f] !== "";

				// Style the dot differently for captures vs normal moves
				dot.className = `absolute bg-blue-500 rounded-full opacity-75 ${
					isCapture ? "z-10" : ""
				}`;

				// Calculate dot size and position
				const ds =
					this.size / 8 / (isCapture ? 2.5 : 4);
				dot.style.width = `${ds}px`;
				dot.style.height = `${ds}px`;
				dot.style.left = `${m.f * (this.size / 8) + (this.size / 8 - ds) / 2}px`;
				dot.style.top = `${m.r * (this.size / 8) + (this.size / 8 - ds) / 2}px`;

				// Style capture dots differently
				if (isCapture) {
					dot.style.backgroundColor =
						"rgba(70, 130, 180, 0.6)";
					dot.style.border =
						"2px solid rgba(255, 255, 255, 0.7)";
				}

				this.boardEl.appendChild(dot);
			});

			// Get the piece image being dragged
			const img = e.currentTarget;
			const rect = img.getBoundingClientRect();

			// Calculate offset from mouse to piece top-left corner
			const ox = e.clientX - rect.left,
				oy = e.clientY - rect.top;

			// Bring piece to front during drag
			img.style.zIndex = 1000;

			// Store drag information
			this.currentDrag = {
				img,
				from: { r, f },
				offsetX: ox,
				offsetY: oy,
			};
		};

		// Handle mouse movement during drag
		_onMouseMove = (e) => {
			if (!this.currentDrag) return;
			const { img, offsetX, offsetY } = this.currentDrag;

			// Calculate new position based on mouse position
			const x =
				e.clientX -
				this.boardEl.getBoundingClientRect().left -
				offsetX;
			const y =
				e.clientY -
				this.boardEl.getBoundingClientRect().top -
				offsetY;

			// Move the piece image
			img.style.left = `${x}px`;
			img.style.top = `${y}px`;
		};

		// Handle mouse up (end of drag)
		_onMouseUp = (e) => {
			if (!this.currentDrag) return;

			// Redraw board to clear any move indicators
			this.generateBoard();

			const { img, from } = this.currentDrag;
			const rect = this.boardEl.getBoundingClientRect();
			const cell = rect.width / 8; // Size of one square

			// Calculate which square the piece was dropped on
			const x = e.clientX - rect.left,
				y = e.clientY - rect.top;
			const to = {
				r: Math.floor(y / cell),
				f: Math.floor(x / cell),
			};

			// Get all legal moves for this piece
			const moves = this.getLegal(from, img.alt);

			// Check if the drop location is a legal move
			if (moves.some((m) => m.r === to.r && m.f === to.f)) {
				this.movePiece(from, to); // Execute the move
			}

			// Reset drag state
			this.currentDrag = null;
		};
	}

	// When the component loads, create a new chess game
	onMount(() => {
		chess = new Chess(board);
		chess.init();
	});
</script>

<!-- The main HTML structure of the chess game -->
<main
	class="flex flex-col items-center justify-center min-h-screen bg-gray-100"
>
	<!-- The chess board container -->
	<div bind:this={board} id="board"></div>

	<!-- Restart game button -->
	<button
		class="mt-4 px-4 py-2 bg-blue-600 text-white rounded hover:bg-blue-700 focus:outline-none"
		on:click={() => chess.restart()}
	>
		Restart Game
	</button>
</main>
