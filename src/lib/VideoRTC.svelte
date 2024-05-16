<script>
	import { onMount, createEventDispatcher } from 'svelte';
	import { browser } from '$app/environment';

	const dispatch = createEventDispatcher();

	const DISCONNECT_TIMEOUT = 5000;
	const RECONNECT_TIMEOUT = 30000;

	const CODECS = [
		'avc1.640029', // H.264 high 4.1 (Chromecast 1st and 2nd Gen)
		'avc1.64002A', // H.264 high 4.2 (Chromecast 3rd Gen)
		'avc1.640033', // H.264 high 5.1 (Chromecast with Google TV)
		'hvc1.1.6.L153.B0', // H.265 main 5.1 (Chromecast Ultra)
		'mp4a.40.2', // AAC LC
		'mp4a.40.5', // AAC HE
		'flac', // FLAC (PCM compatible)
		'opus' // OPUS Chrome, Firefox
	];

	let config = {
		/**
		 * [config] Supported modes (webrtc, webrtc/tcp, mse, hls, mp4, mjpeg).
		 * @type {string}
		 */
		mode: 'webrtc,mse,hls,mjpeg',

		/**
		 * [config] Requested medias (video, audio, microphone).
		 * @type {string}
		 */
		media: 'video,audio',

		/**
		 * [config] Run stream when not displayed on the screen. Default `false`.
		 * @type {boolean}
		 */
		background: false,

		/**
		 * [config] Run stream only when player in the viewport. Stop when user scroll out player.
		 * Value is percentage of visibility from `0` (not visible) to `1` (full visible).
		 * Default `0` - disable;
		 * @type {number}
		 */
		visibilityThreshold: 0,

		/**
		 * [config] Run stream only when browser page on the screen. Stop when user change browser
		 * tab or minimise browser windows.
		 * @type {boolean}
		 */
		visibilityCheck: true,

		/**
		 * [config] WebRTC configuration
		 * @type {RTCConfiguration}
		 */
		pcConfig: {
			iceServers: [{ urls: 'stun:stun.l.google.com:19302' }]
			// sdpSemantics: 'unified-plan',  // important for Chromecast 1
		}
	};

	export let info = {
		/**
		 * [info] WebSocket connection state. Values: CONNECTING, OPEN, CLOSED
		 * @type {number}
		 */
		wsState: 3, //WebSocket.CLOSED;

		/**
		 * [info] WebRTC connection state.
		 * @type {number}
		 */
		pcState: 3 //WebSocket.CLOSED;
	};

	/**
	 * @type {HTMLVideoElement}
	 */
	let video;

	/**
	 * @type {WebSocket | null}
	 */
	let ws = null;

	/**
	 * @type {string}
	 */
	let wsURL = '';

	/**
	 * @type {RTCPeerConnection | null}
	 */
	let pc = null;

	/**
	 * @type {number}
	 */
	let connectTS = 0;

	/**
	 * @type {string}
	 */
	let mseCodecs = '';

	/**
	 * [internal] Disconnect TimeoutID.
	 * @type {number}
	 */
	let disconnectTID = 0;

	/**
	 * [internal] Reconnect TimeoutID.
	 * @type {number}
	 */
	let reconnectTID = 0;

	/**
	 * [internal] Handler for receiving Binary from WebSocket.
	 * @type {Function | null}
	 */
	let ondata = null;

	/**
	 * [internal] Handlers list for receiving JSON from WebSocket.
	 * @type {Object.<string,Function> | null}
	 */
	let onmessage = null;

	/**
	 * Video source (WebSocket URL).
	 * @type {string | URL}
	 */
	export let src = '';
	$: {
		let value = '';
		if (src instanceof URL) {
			value = src.toString();
		} else {
			value = src;
		}

		if (value.startsWith('http')) {
			value = 'ws' + value.substring(4);
		} else if (value.startsWith('/')) {
			value = 'ws' + location.origin.substring(4) + value;
		}

		wsURL = value;
	}

	/**
	 * Play video. Support automute when autoplay blocked.
	 * https://developer.chrome.com/blog/autoplay/
	 */
	function play() {
		video?.play().catch(() => {
			if (!video) return;
			if (!video.muted) {
				video.muted = true;
				video.play().catch((er) => {
					console.warn(er);
				});
			}
		});
	}

	/**
	 * Send message to server via WebSocket
	 * @param {Object} value
	 */
	function send(value) {
		if (ws) ws.send(JSON.stringify(value));
	}

	/** @param {Function} isSupported */
	function codecs(isSupported) {
		return CODECS.filter(
			(codec) => config.media.indexOf(codec.indexOf('vc1') > 0 ? 'video' : 'audio') >= 0
		)
			.filter((codec) => isSupported(`video/mp4; codecs="${codec}"`))
			.join();
	}

	/**
	 * `CustomElement`. Invoked each time the custom element is appended into a
	 * document-connected element.
	 */
	function connectedCallback() {
		if (disconnectTID) {
			clearTimeout(disconnectTID);
			disconnectTID = 0;
		}

		// because video autopause on disconnected from DOM
		if (video) {
			const seek = video.seekable;
			if (seek.length > 0) {
				video.currentTime = seek.end(seek.length - 1);
			}
			play();
		} else {
			oninit();
		}

		onconnect();
	}

	/**
	 * `CustomElement`. Invoked each time the custom element is disconnected from the
	 * document's DOM.
	 */
	function disconnectedCallback() {
		if (config.background || disconnectTID) return;
		if (info.wsState === WebSocket.CLOSED && info.pcState === WebSocket.CLOSED) return;

		disconnectTID = setTimeout(() => {
			if (reconnectTID) {
				clearTimeout(reconnectTID);
				reconnectTID = 0;
			}

			disconnectTID = 0;

			ondisconnect();
		}, DISCONNECT_TIMEOUT);
	}

	/**
	 * Creates child DOM elements. Called automatically once on `connectedCallback`.
	 */
	function oninit() {
		dispatch('init');

		if (!video) {
			console.error('Video element not found');
			return;
		}
		// // video = document.createElement('video');
		// video.controls = true;
		// video.playsInline = true;
		// video.preload = 'auto';

		// video.style.display = 'block'; // fix bottom margin 4px
		// video.style.width = '100%';
		// video.style.height = '100%';

		// appendChild(video);

		// all Safari lies about supported audio codecs
		const m = window.navigator.userAgent.match(/Version\/(\d+).+Safari/);
		if (m) {
			// AAC from v13, FLAC from v14, OPUS - unsupported
			const skip = m[1] < '13' ? 'mp4a.40.2' : m[1] < '14' ? 'flac' : 'opus';
			CODECS.splice(CODECS.indexOf(skip));
		}

		// if (background) return;

		// if ('hidden' in document && visibilityCheck) {
		// 	document.addEventListener('visibilitychange', () => {
		// 		if (document.hidden) {
		// 			disconnectedCallback();
		// 		} else if (isConnected) {
		// 			connectedCallback();
		// 		}
		// 	});
		// }

		// if ('IntersectionObserver' in window && visibilityThreshold) {
		// 	const observer = new IntersectionObserver(
		// 		(entries) => {
		// 			entries.forEach((entry) => {
		// 				if (!entry.isIntersecting) {
		// 					disconnectedCallback();
		// 				} else if (isConnected) {
		// 					connectedCallback();
		// 				}
		// 			});
		// 		},
		// 		{ threshold: visibilityThreshold }
		// 	);
		// 	// observer.observe(this);
		// }
		connectedCallback();
	}

	/**
	 * Connect to WebSocket. Called automatically on `connectedCallback`.
	 * @return {boolean} true if the connection has started.
	 */
	function onconnect() {
		dispatch('connect');

		if (!wsURL || ws || pc) return false;

		// CLOSED or CONNECTING => CONNECTING
		info.wsState = WebSocket.CONNECTING;

		connectTS = Date.now();

		ws = new WebSocket(wsURL);
		ws.binaryType = 'arraybuffer';
		ws.addEventListener('open', () => onopen());
		ws.addEventListener('close', () => onclose());

		dispatch('mode', 'loading');
		return true;
	}

	function ondisconnect() {
		dispatch('disconnect');

		info.wsState = WebSocket.CLOSED;
		if (ws) {
			ws.close();
			ws = null;
		}

		info.pcState = WebSocket.CLOSED;
		if (pc) {
			pc.getSenders().forEach((sender) => {
				if (sender.track) sender.track.stop();
			});
			pc.close();
			pc = null;
		}

		if (!video) return;
		video.src = '';
		video.srcObject = null;
	}

	/**
	 * @returns {Array.<string>} of modes (mse, webrtc, etc.)
	 */
	function onopen() {
		dispatch('open');

		// CONNECTING => OPEN
		info.wsState = WebSocket.OPEN;

		if (!ws) return [];

		ws.addEventListener('message', (ev) => {
			if (typeof ev.data === 'string') {
				const msg = JSON.parse(ev.data);
				for (const mode in onmessage) {
					onmessage[mode](msg);
				}
			} else {
				if (ondata) ondata(ev.data);
			}
		});

		ondata = null;
		onmessage = {};

		/**
		 *  @type {string[]}
		 */
		const modes = [];

		if (
			config.mode.indexOf('mse') >= 0 &&
			('MediaSource' in window || 'ManagedMediaSource' in window)
		) {
			modes.push('mse');
			onmse();
		} else if (
			config.mode.indexOf('hls') >= 0 &&
			video?.canPlayType('application/vnd.apple.mpegurl')
		) {
			modes.push('hls');
			onhls();
		} else if (config.mode.indexOf('mp4') >= 0) {
			modes.push('mp4');
			onmp4();
		}

		if (config.mode.indexOf('webrtc') >= 0 && 'RTCPeerConnection' in window) {
			modes.push('webrtc');
			onwebrtc();
		}

		if (config.mode.indexOf('mjpeg') >= 0) {
			if (modes.length) {
				onmessage['mjpeg'] = (/** @type {{ type: string; value: string | string[]; }} */ msg) => {
					if (msg.type !== 'error' || msg.value.indexOf(modes[0]) !== 0) return;
					onmjpeg();
				};
			} else {
				modes.push('mjpeg');
				onmjpeg();
			}
		}

		onmessage['stream'] = (/** @type {{ type: string; value: any; }} */ msg) => {
			dispatch('message', msg);
			switch (msg.type) {
				case 'error':
					// statusError = msg.value;
					dispatch('error', msg.value);
					break;
				case 'mse':
				case 'hls':
				case 'mp4':
				case 'mjpeg':
					// statusMode = msg.type.toUpperCase();
					dispatch('mode', msg.type);
					break;
			}
		};

		return modes;
	}

	/**
	 * @return {boolean} true if reconnection has started.
	 */
	function onclose() {
		dispatch('close');

		if (info.wsState === WebSocket.CLOSED) return false;

		// CONNECTING, OPEN => CONNECTING
		info.wsState = WebSocket.CONNECTING;
		ws = null;

		// reconnect no more than once every X seconds
		const delay = Math.max(RECONNECT_TIMEOUT - (Date.now() - connectTS), 0);

		reconnectTID = setTimeout(() => {
			reconnectTID = 0;
			onconnect();
		}, delay);

		return true;
	}

	function onmse() {
		/** @type {MediaSource} */
		let ms;

		if ('ManagedMediaSource' in window) {
			// const MediaSource = window.ManagthisedMediaSource; // TODO: check what's going on here

			ms = new MediaSource();
			ms.addEventListener(
				'sourceopen',
				() => {
					send({ type: 'mse', value: codecs(MediaSource.isTypeSupported) });
				},
				{ once: true }
			);
			if (video) {
				video.disableRemotePlayback = true;
				video.srcObject = ms;
			}
		} else {
			ms = new MediaSource();
			ms.addEventListener(
				'sourceopen',
				() => {
					if (video) URL.revokeObjectURL(video.src);
					send({ type: 'mse', value: codecs(MediaSource.isTypeSupported) });
				},
				{ once: true }
			);

			if (video) {
				video.src = URL.createObjectURL(ms);
				video.srcObject = null;
			}
		}

		play();

		mseCodecs = '';

		if (!onmessage) {
			console.error('VideoRTC.onmse: no onmessage');
			return;
		}
		onmessage['mse'] = (/** @type {{ type: string; value: string; }} */ msg) => {
			if (msg.type !== 'mse') return;

			mseCodecs = msg.value;

			const sb = ms.addSourceBuffer(msg.value);
			sb.mode = 'segments'; // segments or sequence
			sb.addEventListener('updateend', () => {
				if (sb.updating) return;

				try {
					if (bufLen > 0) {
						const data = buf.slice(0, bufLen);
						bufLen = 0;
						sb.appendBuffer(data);
					} else if (sb.buffered && sb.buffered.length) {
						const end = sb.buffered.end(sb.buffered.length - 1) - 15;
						const start = sb.buffered.start(0);
						if (end > start) {
							sb.remove(start, end);
							ms.setLiveSeekableRange(end, end + 15);
						}
					}
				} catch (e) {
					console.error(e);
				}
			});

			const buf = new Uint8Array(2 * 1024 * 1024);
			let bufLen = 0;

			ondata = (/** @type {ArrayBuffer} */ data) => {
				if (sb.updating || bufLen > 0) {
					const b = new Uint8Array(data);
					buf.set(b, bufLen);
					bufLen += b.byteLength;
				} else {
					try {
						sb.appendBuffer(data);
					} catch (e) {
						console.error(e);
					}
				}
			};
		};
	}

	function onwebrtc() {
		/**
		 * @type {RTCPeerConnection | null}
		 */
		let pc = new RTCPeerConnection(config.pcConfig);

		pc.addEventListener('icecandidate', (ev) => {
			if (ev.candidate && config.mode.indexOf('webrtc/tcp') >= 0 && ev.candidate.protocol === 'udp')
				return;

			const candidate = ev.candidate ? ev.candidate.toJSON().candidate : '';
			send({ type: 'webrtc/candidate', value: candidate });
		});

		pc.addEventListener('connectionstatechange', () => {
			if (pc?.connectionState === 'connected') {
				const tracks = pc.getReceivers().map((receiver) => receiver.track);
				/** @type {HTMLVideoElement} */
				const video2 = document.createElement('video');
				video2.addEventListener('loadeddata', () => onpcvideo(video2), { once: true });
				video2.srcObject = new MediaStream(tracks);
			} else if (pc?.connectionState === 'failed' || pc?.connectionState === 'disconnected') {
				pc.close(); // stop next events

				info.pcState = WebSocket.CLOSED;
				pc = null;

				onconnect();
			}
		});

		if (!onmessage) return;
		onmessage['webrtc'] = (/** @type {{ type: any; value: string; }} */ msg) => {
			switch (msg.type) {
				case 'webrtc/candidate':
					if (config.mode.indexOf('webrtc/tcp') >= 0 && msg.value.indexOf(' udp ') > 0) return;

					pc?.addIceCandidate({ candidate: msg.value, sdpMid: '0' }).catch((er) => {
						console.warn(er);
					});
					break;
				case 'webrtc/answer':
					pc?.setRemoteDescription({ type: 'answer', sdp: msg.value }).catch((er) => {
						console.warn(er);
					});
					break;
				case 'error':
					if (msg.value.indexOf('webrtc/offer') < 0) return;
					pc?.close();
			}
		};

		createOffer(pc).then((offer) => {
			send({ type: 'webrtc/offer', value: offer.sdp });
		});

		info.pcState = WebSocket.CONNECTING;
		pc = pc;
	}

	/**
	 * @param pc {RTCPeerConnection}
	 * @return {Promise<RTCSessionDescriptionInit>}
	 */
	async function createOffer(pc) {
		try {
			if (config.media.indexOf('microphone') >= 0) {
				const media = await navigator.mediaDevices.getUserMedia({ audio: true });
				media.getTracks().forEach((track) => {
					pc.addTransceiver(track, { direction: 'sendonly' });
				});
			}
		} catch (e) {
			console.warn(e);
		}

		for (const kind of ['video', 'audio']) {
			if (config.media.indexOf(kind) >= 0) {
				pc.addTransceiver(kind, { direction: 'recvonly' });
			}
		}

		const offer = await pc.createOffer();
		await pc.setLocalDescription(offer);
		return offer;
	}

	/**
	 * @param video2 {HTMLVideoElement}
	 */
	function onpcvideo(video2) {
		if (pc) {
			// Video+Audio > Video, H265 > H264, Video > Audio, WebRTC > MSE
			let rtcPriority = 0,
				msePriority = 0;

			if (!video2.srcObject) {
				console.error('VideoRTC.onpcvideo: no srcObject');
				return;
			}
			if (!video) {
				console.error('VideoRTC.onpcvideo: no video');
				return;
			}

			/** @type {MediaStream} */
			// @ts-ignore
			const stream = video2.srcObject;
			if (stream.getVideoTracks().length > 0) rtcPriority += 0x220;
			if (stream.getAudioTracks().length > 0) rtcPriority += 0x102;

			if (mseCodecs.indexOf('hvc1.') >= 0) msePriority += 0x230;
			if (mseCodecs.indexOf('avc1.') >= 0) msePriority += 0x210;
			if (mseCodecs.indexOf('mp4a.') >= 0) msePriority += 0x101;

			if (rtcPriority >= msePriority) {
				video.srcObject = stream;
				play();

				info.pcState = WebSocket.OPEN;

				info.wsState = WebSocket.CLOSED;
				if (ws) {
					ws.close();
					ws = null;
				}
			} else {
				info.pcState = WebSocket.CLOSED;
				if (pc) {
					pc.close();
					pc = null;
				}
			}
		}

		video2.srcObject = null;

		if (info.pcState !== WebSocket.CLOSED) {
			dispatch('mode', 'rtc');
		}
	}

	function onmjpeg() {
		ondata = (/** @type {ArrayBuffer} */ data) => {
			if (!video) return;
			video.controls = false;
			video.poster = 'data:image/jpeg;base64,' + btoa(data);
		};

		send({ type: 'mjpeg' });
	}

	function onhls() {
		if (!onmessage) {
			console.error('VideoRTC.onhls: no onmessage');
			return;
		}
		onmessage['hls'] = (/** @type {{ type: string; value: string; }} */ msg) => {
			if (msg.type !== 'hls') return;

			const url = 'http' + wsURL.substring(2, wsURL.indexOf('/ws')) + '/hls/';
			const playlist = msg.value.replace('hls/', url);
			if (!video) {
				console.error('VideoRTC.onhls: no video');
				return;
			}
			video.src = 'data:application/vnd.apple.mpegurl;base64,' + btoa(playlist);
			play();
		};

		send({ type: 'hls', value: codecs((/** @type {string} */ type) => video?.canPlayType(type)) });
	}

	function onmp4() {
		/** @type {HTMLCanvasElement} **/
		const canvas = document.createElement('canvas');
		/** @type {CanvasRenderingContext2D | null} */
		let context;

		/** @type {HTMLVideoElement} */
		const video2 = document.createElement('video');
		video2.autoplay = true;
		video2.playsInline = true;
		video2.muted = true;

		video2.addEventListener('loadeddata', () => {
			if (!context) {
				canvas.width = video2.videoWidth;
				canvas.height = video2.videoHeight;
				context = canvas.getContext('2d');
			}

			context?.drawImage(video2, 0, 0, canvas.width, canvas.height);

			if (!video) {
				console.error('VideoRTC.onmp4: no video');
				return;
			}
			video.controls = false;
			video.poster = canvas.toDataURL('image/jpeg');
		});

		ondata = (/** @type {ArrayBuffer} */ data) => {
			video2.src = 'data:video/mp4;base64,' + btoa(data);
		};

		if (!video) {
			console.error('VideoRTC.onmp4: no video');
			return;
		}
		send({ type: 'mp4', value: codecs(video.canPlayType) });
	}

	/**
	 * @param input {ArrayBuffer | string}
	 */
	function btoa(input) {
		if (typeof input === 'string') return window.btoa(input);

		const bytes = new Uint8Array(input);
		const len = bytes.byteLength;
		let binary = '';
		for (let i = 0; i < len; i++) {
			binary += String.fromCharCode(bytes[i]);
		}
		return window.btoa(binary);
	}

	onMount(() => {
		if (browser && video) {
			console.log('VideoRTC.onMount: video', video);
			oninit();
		}
	});
</script>

<video bind:this={video} {...$$props}>
	<track kind="captions" />
</video>
