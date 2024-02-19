## youtube-shorts-doc

1 - gerar uma api key do youtube: https://developers.google.com/youtube/v3?hl=pt-br<br>
 1.1 - no back na variavel $KEY colocar sua key gerada:  $KEY = '[COLOCAR SUA KEY]';

 2 - criar uma playlist no youtube que terão todos os shorts 
   2.1 - chamar essa função com o id da playlist com shorts
   
    ```
    getYoutubeVideosByChannelIdOrPlaylistId({
			id: 'PLaGX332bqyauM43qwd-LsQJ1zrq2Vmp7m', //PLAYLIST-ID
			type: 'playlistItems',
			maxResults: 40
		})
    ```

# container
 ```
<div class="swiper swiper-youtube-shorts">
			<div class="swiper-wrapper">
			</div>
	</div>
 ```

# back
 ```
<?php
function validateScriptExecution($params)
{
	// Validate SQL injection
	$sqlInjectionKeywords = array(
		"SELECT", "INSERT", "UPDATE", "DELETE", "DROP", "UNION", "OR", "AND", "FROM"
	);
	foreach ($params as $param) {

		foreach ($sqlInjectionKeywords as $keyword) {
			if (stripos(mb_strtoupper($param), $keyword) !== false) {
				// SQL injection detected
				return false;
			}
		}

		// Validate file name
		if (!preg_match('/^[a-zA-Z0-9_-]+$/', $param)) {
			// Invalid file name format
			return false;
		}
	}

	// All validations passed
	return true;
}

function consultarApi($url, $params) {
	$queryParams = http_build_query($params);

	$ch = curl_init();
	curl_setopt($ch, CURLOPT_URL, $url . '?' . $queryParams);

	curl_setopt($ch, CURLOPT_HEADER, false);
	curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
	curl_setopt($ch, CURLOPT_FOLLOWLOCATION, true);

	curl_setopt($ch, CURLOPT_TIMEOUT, 10);
	curl_setopt($ch, CURLOPT_SSL_VERIFYPEER, false);
	curl_setopt($ch, CURLOPT_SSL_VERIFYHOST, false);

	$resp = curl_exec($ch);

	curl_close($ch);

	$resp_invalida = false;

	if ($resp === false) {
		$resp_invalida = true;
	}

	if (!$resp_invalida) {
		$resp = json_decode($resp, true);
		if ($resp === null && json_last_error() !== JSON_ERROR_NONE) {
			$resp_invalida = true;
		}
	}

	return !$resp_invalida ? $resp : false;
}

function lerArquivo($file_name) {
	if (!file_exists($file_name)) return [];

	$handle = fopen($file_name, 'r');
	$contents = fread($handle, filesize($file_name));
	fclose($handle);

	return json_decode($contents, true);
}

function escreverArquivo($file_name, $dado) {
	$handle = fopen($file_name, 'w');
	fwrite($handle, is_array($dado) ? json_encode($dado) : $dado);
	fclose($handle);
}

function consultarTodosItemsDaPlaylist($url, $params) {
	$file_name_base = 'cache/videos-youtube/playlist_' . $params['playlistId'] . '_page_';

	$file = $file_name_base . '1.txt';

	$url = 'https://www.googleapis.com/youtube/v3/playlistItems';

	// buscando primeira página
	$respPrimeiraPagina = consultarApi($url, $params);

	if(!$respPrimeiraPagina) return false;

	$file = $file_name_base . 'last_total_items.txt';

	$dados_total_pagina = lerArquivo($file);

	$total_items = isset($dados_total_pagina['totalResults']) ? $dados_total_pagina['totalResults'] : 0;

	$total_items_atual = $respPrimeiraPagina['pageInfo']['totalResults'];

	escreverArquivo($file, [
		'totalResults' => $total_items_atual
	]);

	$total_diferente = $total_items != $total_items_atual;

	while(isset($respPrimeiraPagina['nextPageToken']) && $respPrimeiraPagina['nextPageToken']) {
		$params['pageToken'] = $respPrimeiraPagina['nextPageToken'];

		unset($respPrimeiraPagina['nextPageToken']);

		$file = $file_name_base . $params['pageToken'] . '.txt';
	
		if ($total_diferente || !file_exists($file)) {

			$resp = consultarApi($url, $params);

			if($resp && $resp['items']) {
				escreverArquivo($file, $resp);
			}
				
		} else {
			$resp = lerArquivo($file);
		}

		if($resp && $resp['items']) {

			if($resp['nextPageToken'])
				$respPrimeiraPagina['nextPageToken'] = $resp['nextPageToken'];

			$respPrimeiraPagina['items'] = array_merge($respPrimeiraPagina['items'], $resp['items']);
		}
	}

	return $respPrimeiraPagina;
}

function filtrarUltimosVideosDaPlaylistCompleta($playlist_completa, $total_videos_pegar) {
	if(!isset($playlist_completa['items']) || count($playlist_completa['items']) == 0 || !is_numeric($total_videos_pegar)) return [];

	$copia_playlist = $playlist_completa;

	$posicao_inicial = count($playlist_completa['items']) - $total_videos_pegar - 1;

	$posicao_inicial = $posicao_inicial > 0 ? $posicao_inicial : 0;

	$copia_playlist['items'] = array_slice($playlist_completa['items'], $posicao_inicial);

	return $copia_playlist;
}

header('Content-Type:application/json');

//COLOCAR SUA API KEY DO YOUTUBE AQUI
$KEY = '[COLOCAR SUA KEY]';

$params = $_GET;

if (!validateScriptExecution($params)) return '';

$params['key'] = $KEY;

$queryParams = http_build_query($params);

$url = '';
$dados = [];

if (isset($params['channelId'])) {
	$tempo = 10800;
	$file = 'cache/videos-youtube/videos_canal_' . $params['channelId'] . '.txt';
	$url = 'https://www.googleapis.com/youtube/v3/search';

	if (!file_exists($file) || ($tempo > 0 && filemtime($file) < time() - $tempo)) {

		$resp = consultarApi($url, $params);
	
		if($resp) {
			escreverArquivo($file, $resp);
			$dados = $resp;
		} else {
			$dados = lerArquivo($file);
			escreverArquivo($file, $dados); // reescrevendo o arquivo para mudar data de modificação
		}
	
	} else {
		$dados = lerArquivo($file);
	}
}

if (isset($params['playlistId'])) {
	$tempo = 10800;
	$file = 'cache/videos-youtube/playlist_' . $params['playlistId'] . '_complete.txt';
	$url = 'https://www.googleapis.com/youtube/v3/playlistItems';
	
	$total_items_retornar = isset($params['maxResults']) ? $params['maxResults'] : 40;
	$params['maxResults'] = 50; // sempre buscar o máximo que conseguir da api e fazer paginação falsa

	if (!file_exists($file) || ($tempo > 0 && filemtime($file) < time() - $tempo)) {

		$resp = consultarTodosItemsDaPlaylist($url, $params);
	
		if($resp) {
			escreverArquivo($file, $resp);
			$dados = $resp;
		} else {
			$dados = lerArquivo($file);
			escreverArquivo($file, $dados); // reescrevendo o arquivo para mudar data de modificação
		}
	
	} else {
		$dados = lerArquivo($file);
	}

	$dados = filtrarUltimosVideosDaPlaylistCompleta($dados, $total_items_retornar);
}

http_response_code(200);
echo json_encode($dados);
```

# front
```
function mountYoutubeHtmlCards(videoData, channel_id) {
			const changeChannelName = {
				'Bruno Brito': '<?= tt('Dicas para montar sua loja') ?>'
			}
			const container = document.querySelector('.youtube--section')
			let html = ''
			let titleHtml = ''
			videoData.forEach((video, i) => {
				if (i == 0) titleHtml += `
					<div class="d-flex" style="justify-content: space-between;">
						<strong style="font-size: 12px;font-weight: 900;display: flex;align-items: center;"><strong style="font-weight: 700;margin-right: 8px;font-size: 16px;">Youtube </strong></strong>
						<a href="http://youtube.com/channel/${channel_id}/?sub_confirmation=1" target="_blank" style="text-decoration: none;font-weight: 600;color: gray;"><?= tt('Inscreva-se') ?> -></a>
					</div>`

				if (!video.snippet.title.toLowerCase().includes("#shorts")) {
					html += `
					<a class="swiper-slide" href="http://youtube.com/video/${video.id.videoId}" target="_blank"><img src="${video.snippet.thumbnails.high.url}"></a>
					`
				} //CHECK IF THE VIDEO IS A SHORT

			})

			container.insertAdjacentHTML('afterbegin', `
			<div class="youtube--content">
				${titleHtml}
			</div>`)
			document.querySelector('.swiper-youtube-videos .swiper-wrapper').insertAdjacentHTML('afterbegin', html)
		}

		function mountYoutubeShorts(shorts) {
			const container = document.querySelector('.swiper-youtube-shorts .swiper-wrapper')
			let html = ''
			shorts.forEach((short, i) => {
				if (!short.snippet.thumbnails?.default?.url) return

				let shortImage = short.snippet.thumbnails.default.url
				html += `
				<div class="swiper-slide swiper-slide-shorts-youtube" style="width: 66px;cursor: pointer;" data-youtube-short-link="${short.snippet.resourceId.videoId}" data-youtube-short-title="${short.snippet.title}"  data-bs-toggle="modal" data-bs-target="#youtubeShortsModal" data-toggle="tooltip" data-bs-placement="bottom" title="${short.snippet.title}">
					<img width="66px" height="66px" src="${shortImage}" style="object-fit:cover;border-radius: 50%;" />
				</div>
				`
			})

			container.insertAdjacentHTML('afterbegin', html)

			initTooltip("[data-toggle='tooltip']")
		}

		async function getYoutubeVideosByChannelIdOrPlaylistId({
			id,
			type,
			maxResults
		}) {

			try {

				let dinamicParams = {
					search: `&channelId=${id}`,
					playlistItems: `&playlistId=${id}`,
				}

				const URL = `<?= base_url('/buscarVideosYoutube.php') ?>?${dinamicParams[type]}&order=date&part=snippet&maxResults=${maxResults}`
				const resYoutube = await fetch(URL)
				const data = await resYoutube.json()
				if (type === 'search') {

					const videos = data.items
					mountYoutubeHtmlCards(videos, id)
				} else {
					const shorts = data.items.reverse()
					mountYoutubeShorts(shorts)
				}
			} catch (err) {
				console.error({
					err
				})
			}
		}

		getYoutubeVideosByChannelIdOrPlaylistId({
			id: 'PLaGX332bqyauM43qwd-LsQJ1zrq2Vmp7m', //PLAYLIST-ID
			type: 'playlistItems',
			maxResults: 40
		})


		function mountShortsModal() {
			const shortsModal = document.getElementById('youtubeShortsModal')
			const shortsModalBody = document.querySelector('.modal-body-youtube-shorts')
			shortsModal.addEventListener('show.bs.modal', function(event) {
				// Button that triggered the modal
				const button = event.relatedTarget
				const link = button.getAttribute('data-youtube-short-link')
				const title = button.getAttribute('data-youtube-short-title')

				shortsModalBody.innerHTML = `
				<div class="short--iframe">
					<button type="button" class="btn-close" data-bs-dismiss="modal" aria-label="Close">X</button>
					<iframe style="width: 100%; height: 85vh;border-radius: 20px;" src="https://www.youtube.com/embed/${link}?autoplay=1&mute=0" title="${title}" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>
				</div>
				`
			})

			shortsModal.addEventListener('hide.bs.modal', function(event) {
				shortsModalBody.innerHTML = '' // This will remove the iframe and stop the video
			})
		}
		mountShortsModal()
```

# css

```
.swiper-youtube-shorts {
  width: 100%;
  margin-bottom: 24px;
}
.swiper-youtube-shorts .swiper-slide-shorts-youtube::before {
  content: " ";
  position: absolute;
  top: 0px;
  left: 0px;
  right: 0px;
  bottom: 0px;
  border: 5px solid #F4F6F8;
  border-radius: 50%;
}
.swiper-youtube-shorts .swiper-slide-shorts-youtube::after {
  content: " ";
  position: absolute;
  top: 0px;
  left: 0px;
  right: 0px;
  bottom: 0px;
  border: 2px solid #0A0226;
  border-radius: 50%;
}
```

# instancia do swiper

```
new Swiper('.swiper-youtube-shorts', {
		slidesPerView: "auto",
		spaceBetween: 16,
		slidesPerGroup: 3,
	});
```

dependencias:
  - bootstrap
  - swiper slider https://swiperjs.com/get-started
