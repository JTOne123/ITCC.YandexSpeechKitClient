# ITCC Yandex SpeechKit client

[![NuGet version](https://badge.fury.io/nu/ITCC.YandexSpeechKitClient.svg)](https://badge.fury.io/nu/ITCC.YandexSpeechKitClient)

## General

[The SpeechKit Cloud API](https://tech.yandex.com/speechkit/cloud/) is an HTTP API that allows application developers to use Yandex speech technologies. This library implements api client for .NET 4.5+ and .NET Standard 1.3+.

## Capabilities

* Speech recognition ([HTTP mode](https://tech.yandex.com/speechkit/cloud/doc/guide/concepts/asr-http-request-docpage/) / [Data Streaming mode](https://tech.yandex.com/speechkit/cloud/doc/guide/concepts/asr-protobuf-docpage)). Supported languages:
    * Russian  
    * English  
    * Ukrainian  
    * Turkish  
    * German  
    * Spanish  
    * French  
* Speech synthesis. Supported languages:  
    * Russian  
    * English  
    * Ukrainian  
    * Turkish  
    
Note that some speech models are not available for all languages.

## Usage

Use `SpeechKitClientOptions` class to configure basic API access parameters:
```c#
var apiSetttings = new SpeechKitClientOptions("apiKey", "someApplication", Guid.Empty, "device");
```

### Speech recognition

Automatic speech recognition is the process of turning speech into text. Yandex SpeechKit Cloud makes it possible to recognize spontaneous speech in multiple languages.

#### HTTP mode

Use HTTP mode for simple speech-to-text conversion if you want just convert one audio file to text without intermediate results or analytics:

```c#
using (var client = new SpeechKitClient(apiSetttings))
{
    var speechRecognitionOptions = new SpeechRecognitionOptions(SpeechModel.Queries, RecognitionAudioFormat.Pcm16K, RecognitionLanguage.Russian);
    try
    {
        var result = await client.SpeechToTextAsync(speechRecognitionOptions, mediaStream, cancellationToken).ConfigureAwait(false);
        if (result.TransportStatus != TransportStatus.Ok || result.StatusCode != HttpStatusCode.OK)
        {
            //Handle network and request parameters error
        }

        if (!result.Result.Success)
        {
            //Unable to recognize speech
        }

        var utterances = result.Result.Variants;
        //Use recognition results

    }
    catch (OperationCanceledException)
    {
        //Handle operation cancellation
    }
}
```

**[Note]** Audio data size limit by **1 MB**.

Speech recognition begins only after all the audio data has been transmitted to the server.

#### Data streaming mode

If you want to receive intermediate result during recognition or advanced information (skeaker age group, detected language, recognition confidence of each word in phrase etc.) - use data streaming mode.

First of all, start new speech recognition session:

```c#
var sessionOptions = new SpeechRecognitionSessionOptions(SpeechModel.Queries, RecognitionAudioFormat.Pcm16K)
{
    Language = RecognitionLanguage.Russian,
    BiometryParameters = BiometryParameters.Gender | BiometryParameters.Group,
    Position = new Position(latitude, longitude)
};

try
{
    var startSessionResult = await SpeechRecognitionSessionFactory.CreateNewSpeechRecognitionSessionAsync(apiSetttingse, sessionOptions, token).ConfigureAwait(false);
    switch (startSessionResult.TransportStatus)
    {
        case TransportStatus.Ok:
            //Session started, use startSessionResult.Session
            break;
        case TransportStatus.UnexpectedServerResponse:
            //This means API was changed.
            return;
        case TransportStatus.SslNegotiationError:
            //Unable to create ssl connection - possible ssl certificate substitution.
            return;
        case TransportStatus.Timeout:
            //Operation timed out. Timeout settings can be configured in client.
            return;
        case UnexpectedEndOfMessage:
            //Server response message suddenly ended and was't successfully parsed. Try again.
            return;
        case TransportStatus.SocketError:
            //Some network error occured. Use startSessionResult.SocketError
            return;
        default:
            throw new ArgumentOutOfRangeException();
    }
}
catch (OperationCanceledException)
{
    //Handle operation cancellation
}
```

If session start fails, session sould be disposed. Check parameters and retry attempt.

If session successfully started, `session.SendChunkAsync` and `session.ReceiveRecognitionResultAsync` could be used.

To send data chunk, use
```c#
var sendChunkResult = await session.SendChunkAsync(audioData, lastChunk, cancellationToken).ConfigureAwait(false);
```

Verify operation result using ``sendChunkResult.TransportStatus``.

Use `lastChunk = true` when sending final data chunk. In this case server sends final recognition results and closes connection.

To receive recognition results, use:
```c#
var getResponseResult = await session.ReceiveRecognitionResultAsync(cancellationToken).ConfigureAwait(false);
```

Several data chunks could be sent before results will be requested. In this case chunk recognition results of the same utterance will be merged into single response. Recognition results of different utterances always sends in separate messages.

### Speech synthesis

Speech synthesis (text-to-speech) is the process of generating speech from printed text. Yandex SpeechKit Cloud can produce speech for any texts in several languages. You can also choose the voice (male or female) and the intonation.

To convert text to speech, use:

```c#
using (var client = new SpeechKitClient(apiSetttings))
{
    var options = new SynthesisOptions("Text to be spoken", 1.4)
    {
        AudioFormat = SynthesisAudioFormat.Wav,
        Language = SynthesisLanguage.English,
        Emotion = Emotion.Neutral,
        Quality = SynthesisQuality.High,
        Speaker = Speaker.Alyss
    };

    using (var textToSpechResult = await client.TextToSpeechAsync(options, cancellationToken).ConfigureAwait(false))
    {
        if (textToSpechResult.TransportStatus != TransportStatus.Ok || textToSpechResult.ResponseCode != HttpStatusCode.OK)
        {
            //handle network and request parameters errors
        }

        // textToSpechResult.Result contains generated audio data
    }
}
```

Default `speed` parameter value is `1.0` - average rate of human speech. You canchange speed in range from `0.1` to `3.0`.
