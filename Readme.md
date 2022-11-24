# Biblioteca de Voz Alexa

=======

*Uma biblioteca e aplicativo de exemplo para abstrair o acesso ao serviço Amazon Alexa para aplicativos Android.*

Em primeiro lugar, meu objetivo com este projeto é ajudar outras pessoas que têm menos conhecimento de Java, Android ou ambos, a serem capazes de integrar rápida e facilmente a plataforma Amazon Alexa em seus próprios aplicativos.

Para começar a usar a plataforma Amazon Alexa, dê uma olhada aqui: https://developer.amazon.com/appsandservices/solutions/alexa/alexa-voice-service/getting-started-with-the-alexa-voice-service

Biblioteca atualizada com funcionalidade para Alexa [versão da API v20160207](https://developer.amazon.com/public/solutions/alexa/alexa-voice-service/content/avs-api-overview).

## Formulários

### Aplicativo de amostra

Curioso sobre o que a biblioteca pode fazer? Um exemplo rápido das três funções principais (eventos de áudio gravados ao vivo, intenções de conversão de texto em fala, intenções de áudio pré-gravadas), além de código de exemplo, pode ser encontrado no [aplicativo de exemplo](https://play.google.com/store /apps/details?id=com.willblaschko.android.alexavoicelibrary)

#### Compilando e executando o aplicativo de exemplo
* Siga o processo para criar um dispositivo conectado detalhado no link da Amazon na parte superior do Leiame.
* Adicione seu arquivo api_key.txt (parte do processo Amazon) à pasta app/src/main/assets
* Altere [PRODUCT_ID](https://github.com/willblaschko/AlexaAndroid/blob/master/app/src/main/java/com/willblaschko/android/alexavoicelibrary/global/Constants.java#L8) para o valor de meu aplicativo Alexa que você configurou para "Id do tipo de aplicativo" acima.
* Crie e execute o aplicativo de amostra usando Gradle na linha de comando ou no Android Studio!

### Aplicação de produção

Ou veja o que a biblioteca pode fazer quando convertida em um pacote completo, completo com ouvinte sempre ativo opcional: [Alexa Listens](https://play.google.com/store/apps/details?id=com.willblaschko.android .alexalistens)

## Usando a Biblioteca

A maior parte da biblioteca pode ser acessada através do [AlexaManager](http://willblaschko.github.io/AlexaAndroid/com/willblaschko/android/alexa/AlexaManager.html) e [AlexaAudioPlayer](http://willblaschko.github. io/AlexaAndroid/com/willblaschko/android/alexa/avs/AlexaAudioPlayer.html), ambas são singletons.

### Instalação

*Verifique se você está extraindo de jcenter () para seu projeto (build.gradle no nível do projeto):
```java
buildscript {
    repositories {
        jcenter()
    }
    ...
}

allprojects {
    repositories {
        jcenter()
    }
}
```
* Adicione a biblioteca às suas importações (application-level build.gradle):
```java
compile 'com.willblaschko.android.alexa:AlexaAndroid:2.4.2'
```
* Siga o processo para criar um dispositivo conectado detalhado no link da Amazon na parte superior do Leiame.
* Siga as instruções para adicionar sua chave e preparar a atividade Login with Amazon no [guia do projeto Android 'Login with Amazon'](https://developer.amazon.com/public/apis/engage/login-with-amazon/ docs/create_android_project.html)
* Adicione seu arquivo api_key.txt (parte do processo de Login with Amazon detalhado no link acima) à pasta app/src/main/assets.
* Inicie a integração e os testes!

### Instanciação de biblioteca e análise básica de retorno

```java
private AlexaManager alexaManager;
private AlexaAudioPlayer audioPlayer;
private List<AvsItem> avsQueue = new ArrayList<>();

private void initAlexaAndroid(){
	//obtém nossa instância AlexaManager por conveniência
alexaManager = AlexaManager.getInstance(this, PRODUCT_ID);

//instancia nosso reprodutor de áudio
audioPlayer = AlexaAudioPlayer.getInstance(this);

//Callback para poder remover o item atual e verificar a fila assim que terminarmos de reproduzir um item
audioPlayer.addCallback(alexaAudioPlayerCallback);
}

// Nosso retorno de chamada que lida com a remoção de itens reproduzidos em nosso reprodutor de mídia e, em seguida, verifica se existem mais itens
private AlexaAudioPlayer.Callback alexaAudioPlayerCallback = new AlexaAudioPlayer.Callback() {
	@Override
	public void playerPrepared(AvsItem pendingItem) {

	}

	@Override
	public void itemComplete(AvsItem completedItem) {
		avsQueue.remove(completedItem);
		checkQueue();
	}

	@Override
	public boolean playerError(int what, int extra) {
		return false;
	}

	@Override
	public void dataError(Exception e) {

	}
};

// retorno de chamada assíncrono para comandos enviados ao Alexa Voice
private AsyncCallback<AvsResponse, Exception> requestCallback = new AsyncCallback<AvsResponse, Exception>() {
	@Override
	public void start() {
		//your on start code
	}

	@Override
	public void success(AvsResponse result) {
		Log.i(TAG, "Voice Success");
		handleResponse(result);
	}

	@Override
	public void failure(Exception error) {
		//your on error code
	}

	@Override
	public void complete() {
		 //your on complete code
	}
};

/**
 *Lidar com a resposta enviada de volta da análise do Intent do Alexa, que pode ser qualquer um dos tipos AvsItem (reproduzir, falar, parar, limpar, ouvir)
  * @param resposta uma List<AvsItem> retornada da chamada mAlexaManager.sendTextRequest() em sendVoiceToAlexa()
 */
private void handleResponse(AvsResponse response){
	if(response != null){
		//se tivermos um item de fila limpa na lista, precisamos limpar a fila atual antes de prosseguir
		// iterar para trás para evitar alterar nossas posições de array e obter todos os erros desagradáveis que surgem
		// de fazer isso
		for(int i = response.size() - 1; i >= 0; i--){
			if(response.get(i) instanceof AvsReplaceAllItem || response.get(i) instanceof AvsReplaceEnqueuedItem){
				//limpar nossa fila
				avsQueue.clear();
				//remove item
				response.remove(i);
			}
		}
		avsQueue.addAll(response);
	}
	checkQueue();
}


/**
 *Verifique nossa fila atual de itens e, se tivermos mais para analisar (assim que chegarmos a uma reprodução ou retorno de chamada), prossiga para o
  * próximo item da nossa lista.
  *
  * Estamos lidando com AvsReplaceAllItem em handleResponse() porque ele precisa limpar tudo que está atualmente na fila, antes
  * os novos itens são adicionados à lista, não deve ter função aqui.
  */
private void checkQueue() {

	//se estivermos sem coisas, desligue o telefone e siga em frente
	if (avsQueue.size() == 0) {
		return;
	}

	AvsItem current = avsQueue.get(0);

	if (current instanceof AvsPlayRemoteItem) {
		//reproduzir um URL
		if (!audioPlayer.isPlaying()) {
			audioPlayer.playItem((AvsPlayRemoteItem) current);
		}
	} else if (current instanceof AvsPlayContentItem) {
		//reproduzir um URL
		if (!audioPlayer.isPlaying()) {
			audioPlayer.playItem((AvsPlayContentItem) current);
		}
	} else if (current instanceof AvsSpeakItem) {
		//reproduzir um arquivo de som
		if (!audioPlayer.isPlaying()) {
			audioPlayer.playItem((AvsSpeakItem) current);
		}
	} else if (current instanceof AvsStopItem) {
		//pare nosso jogo
		audioPlayer.stop();
		avsQueue.remove(current);
	} else if (current instanceof AvsReplaceAllItem) {
		audioPlayer.stop();
		avsQueue.remove(current);
	} else if (current instanceof AvsReplaceEnqueuedItem) {
		avsQueue.remove(current);
	} else if (current instanceof AvsExpectSpeechItem) {
		//ouvir a entrada do usuário
		audioPlayer.stop();
		startListening();
	} else if (current instanceof AvsSetVolumeItem) {
		setVolume(((AvsSetVolumeItem) current).getVolume());
		avsQueue.remove(current);
	} else if(current instanceof AvsAdjustVolumeItem){
		adjustVolume(((AvsAdjustVolumeItem) current).getAdjustment());
		avsQueue.remove(current);
	} else if(current instanceof AvsSetMuteItem){
		setMute(((AvsSetMuteItem) current).isMute());
		avsQueue.remove(current);
	}else if(current instanceof AvsMediaPlayCommandItem){
		//falsificar uma pressão de "reproduzir" de hardware
		sendMediaButton(this, KeyEvent.KEYCODE_MEDIA_PLAY);
	}else if(current instanceof AvsMediaPauseCommandItem){
		//fingir um pressionamento de "pausa" de hardware
		sendMediaButton(this, KeyEvent.KEYCODE_MEDIA_PAUSE);
	}else if(current instanceof AvsMediaNextCommandItem){
		//fingir um pressionamento de "próximo" de hardware
		sendMediaButton(this, KeyEvent.KEYCODE_MEDIA_NEXT);
	}else if(current instanceof AvsMediaPreviousCommandItem){
		//fingir um pressionamento "anterior" de hardware
		sendMediaButton(this, KeyEvent.KEYCODE_MEDIA_PREVIOUS);
	}

}

//nossa chamada para começar a ouvir quando recebermos um AvsExpectSpeechItem
protected abstract void startListening();

//ajustar o volume do nosso dispositivo
private void adjustVolume(long adjust){
	setVolume(adjust, true);
}

//definir o volume do nosso dispositivo
private void setVolume(long volume){
	setVolume(volume, false);
}

//definir o volume do nosso dispositivo, lida com ajustar e definir o volume para evitar a repetição do código
private void setVolume(volume longo final, ajuste booleano final){
	AudioManager am = (AudioManager) getSystemService(AUDIO_SERVICE);
	final int max = am.getStreamMaxVolume(AudioManager.STREAM_MUSIC);
	long vol= am.getStreamVolume(AudioManager.STREAM_MUSIC);
	if(adjust){
		vol += volume * max / 100;
	}else{
		vol = volume * max / 100;
	}
	am.setStreamVolume(AudioManager.STREAM_MUSIC, (int) vol, AudioManager.FLAG_VIBRATE);
	//confirm volume change
	alexaManager.sendVolumeChangedEvent(volume, vol == 0, requestCallback);
}

//definir dispositivo para silenciar
private void setMute(final booleano isMute){
	AudioManager am = (AudioManager) getSystemService(AUDIO_SERVICE);
	am.setStreamMute(AudioManager.STREAM_MUSIC, isMute);
	//confirmar dispositivo mudo
	alexaManager.sendMutedEvent(isMute, requestCallback);
}

 private static void sendMediaButton(Context context, int keyCode) {
	KeyEvent keyEvent = new KeyEvent(KeyEvent.ACTION_DOWN, keyCode);
	Intent intent = new Intent(Intent.ACTION_MEDIA_BUTTON);
	intent.putExtra(Intent.EXTRA_KEY_EVENT, keyEvent);
	context.sendOrderedBroadcast(intent, null);

	keyEvent = new KeyEvent(KeyEvent.ACTION_UP, keyCode);
	intent = new Intent(Intent.ACTION_MEDIA_BUTTON);
	intent.putExtra(Intent.EXTRA_KEY_EVENT, keyEvent);
	context.sendOrderedBroadcast(intent, null);
}
```


## Receitas

Aqui está uma rápida visão geral do código que provavelmente será necessário para criar um aplicativo sólido, verifique o [JavaDoc](http://willblaschko.github.io/AlexaAndroid/) para obter mais detalhes.

### Como fazer login na Amazon
```java
//Execute uma verificação assíncrona se estamos logados ou não
alexaManager.checkLoggedIn(mLoggedInCheck);

//Verifique se o usuário já está logado em sua conta Amazon
alexaManager.checkLoggedIn(AsyncCallback...);

//Faça login do usuário
alexaManager.logIn(AuthorizationCallback...);
```

### Enviar áudio ao vivo (bonus: verificar se há [Record Audio](https://developer.android.com/reference/android/Manifest.permission.html#RECORD_AUDIO) permissão)
```java
private final static int MY_PERMISSIONS_REQUEST_RECORD_AUDIO = 1;
private static final int AUDIO_RATE = 16000;
private RawAudioRecorder recorder;
private RecorderView recorderView;

@Override
public void onResume() {
    super.onResume();

    //solicitar permissão API-23 para RECORD_AUDIO
    if (ContextCompat.checkSelfPermission(getActivity(),
            Manifest.permission.RECORD_AUDIO)
            != PackageManager.PERMISSION_GRANTED) {
        if (!ActivityCompat.shouldShowRequestPermissionRationale(getActivity(),
                Manifest.permission.RECORD_AUDIO)) {
            ActivityCompat.requestPermissions(getActivity(),
                    new String[]{Manifest.permission.RECORD_AUDIO},
                    MY_PERMISSIONS_REQUEST_RECORD_AUDIO);
        }
    }
}

//receber solicitação de permissões RECORD_AUDIO
@Override
public void onRequestPermissionsResult(int requestCode,
                                       @NonNull String permissions[],
                                       @NonNull int[] grantResults) {
    switch (requestCode) {
        case MY_PERMISSIONS_REQUEST_RECORD_AUDIO: {
            // If request is cancelled, the result arrays are empty.
            if (!(grantResults.length > 0
                    && grantResults[0] == PackageManager.PERMISSION_GRANTED)){
                getActivity().getSupportFragmentManager().beginTransaction().remove(this).commit();
            }
        }

    }
}

@Override
public void onStop() {
    super.onStop();
    //tear down our recorder on stop
    if(recorder != null){
        recorder.stop();
        recorder.release();
        recorder = null;
    }
}

@Override
public void startListening() {
    if(recorder == null){
        recorder = new RawAudioRecorder(AUDIO_RATE);
    }
    recorder.start();
    alexaManager.sendAudioRequest(requestBody, getRequestCallback());
}

//nosso requestBody de dados de streaming
private DataRequestBody requestBody = new DataRequestBody() {
    @Override
    public void writeTo(BufferedSink sink) throws IOException {
        //enquanto nosso gravador não estiver nulo e ainda estiver gravando, continue gravando nos dados do POST
        while (recorder != null && !recorder.isPausing()) {
            if(recorder != null) {
                final float rmsdb = recorder.getRmsdb();
                if(recorderView != null) {
                    recorderView.post(new Runnable() {
                        @Override
                        public void run() {
                            recorderView.setRmsdbLevel(rmsdb);
                        }
                    });
                }
                if(sink != null && recorder != null) {
                    sink.write(recorder.consumeRecording());
                }
            }

            //dormir e fazer tudo de novo
            try {
                Thread.sleep(25);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        stopListening();
    }

};

//derrubar nosso gravador
private void stopListening(){
    if(recorder != null) {
        recorder.stop();
        recorder.release();
        recorder = null;
    }
}
```

### Enviar áudio pré-gravado
```java
//enviar áudio pré-gravado para Alexa, analisar o retorno de chamada em requestCallback
try {
    //abrir arquivo de ativo
    InputStream is = getActivity().getAssets().open("intros/joke.raw");
    byte[] fileBytes=new byte[is.available()];
    is.read(fileBytes);
    is.close();
    mAlexaManager.sendAudioRequest(fileBytes, getRequestCallback());
} catch (IOException e) {
    e.printStackTrace();
}
```

### Enviar Pedido de Texto
```java
//envie uma solicitação de texto para Alexa, analise o retorno de chamada em requestCallback
mAlexaManager.sendTextRequest(text, requestCallback);
```

### Reproduzir conteúdo devolvido
```java
//Reproduzir um AvsPlayItem retornado por nossas solicitações
audioPlayer.playItem(AvsPlayAudioItem...);

//Reproduzir um AvsSpeakItem retornado por nossas solicitações
audioPlayer.playItem(AvsSpeakItem...);
```

## Todo o resto

Deixe-me saber se você gostaria de contribuir para esta biblioteca!

## Obrigado
[Josh](https://github.com/joshpar) for MP3 work
