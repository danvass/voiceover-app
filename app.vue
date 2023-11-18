<template>
  <UContainer class="mx-auto max-w-6xl">

    <UCard class="mt-10 mb-4">
      <template #header>
        <div class="flex justify-between">
          <h1>Real-time voiceover</h1>
        </div>
      </template>

      <div>Generate a real-time voiceover from your webcam. All you need is an <UTooltip text="You can get it from https://platform.openai.com/api-keys"> OpenAI API Key</UTooltip>, <UTooltip text="You can get it from your profile on ElevenLabs.">ElevenLabs API Key</UTooltip> and <UTooltip text="You can get it from the Get Voices API endpoint from Eleven Labs.">Voice ID</UTooltip>.</div>

      Config:

      <div class="flex items-center gap-1 mt-2 mb-2">
        <UInput type="password" v-model="openaiApiKey" :disabled="isRecording" placeholder="OpenAI API Key" />
        <UInput type="password" v-model="elevenlabsApiKey" :disabled="isRecording" placeholder="ElevenLabs API Key" />
        <UInput type="text" v-model="voiceId" :disabled="isRecording" placeholder="ElevenLabs Voice ID" />
      </div>

      Prompt:
      <div class="mt-2 mb-3">
        <UTextarea v-model="systemPrompt"></UTextarea>
      </div>

      <UButton @click="toggle">{{ isRecording ? "Stop" : "Capture" }}</UButton>

    </UCard>

    <div class="flex flex-wrap">
      <div class="w-full md:w-3/5">
        <h2 class="text-xl font-bold mb-4">Live video</h2>
        <video ref="videoRef" class="w-full border rounded-lg" autoplay></video>
      </div>
      <div class="w-full md:w-2/5">
        <div class="pb-4 px-4">
          <!-- Content for the right column -->
          <h2 class="text-xl font-bold mb-4">Captured scene</h2>
          <img v-if="frame" :src="frame" alt="Captured Frame" class="border rounded-lg mt-4 mb-2" style="max-width: 40%; height: auto;" />
          <audio v-if="frame" ref="audioRef" controls></audio>
          <p class="text-gray-600">
            {{ narrationText || 'Start recording..' }}
          </p>
        </div>
      </div>
    </div>

    <div class="mt-12 text-gray-600">
      This web app only communicates with OpenAI and ElevenLabs. No API keys are stored and will be lost upon refresh.
    </div>
  </UContainer>
</template>

<script setup>
import { onMounted, onUnmounted, ref } from 'vue';
import OpenAI from 'openai';
import axios from 'axios';

let openai;

const videoRef = ref(null);
const audioRef = ref(null);
let voiceId = ref('');
let systemPrompt = ref(`You are Sir David Attenborough. Narrate the picture of the human as if it is a nature documentary.
Make it snarky and funny. Don't repeat yourself. Make it short. If I do anything remotely interesting, make a big deal about it!`);
let isRecording = ref(false);

let frame = ref(null);
let narrationText = ref('');
let narrationAudio = ref(null);

let videoStream;

onMounted(() => {
  startCamera();
});

onUnmounted(() => {
  stopCamera();
});

const toggle = () => {
  if (openaiApiKey.value === '' || elevenlabsApiKey.value === '') {
    alert('Please enter your API keys');
    return;
  }

  openai = new OpenAI({
    apiKey: openaiApiKey.value,
    dangerouslyAllowBrowser: true
  });

  if (!isRecording.value) {
    // Start capturing frames every 2 seconds
   captureFrame(videoRef.value);
  }

  isRecording.value = !isRecording.value;
  console.log('isRecording', isRecording.value);
};

const startCamera = async () => {
  try {
    videoStream = await navigator.mediaDevices.getUserMedia({ video: true });

    if (videoStream) {
      const videoElement = videoRef.value;
      videoElement.srcObject = videoStream;
    }
  } catch (error) {
    console.error('Error accessing the camera:', error);
  }
};

const stopCamera = () => {
  if (videoStream) {
    const tracks = videoStream.getTracks();
    tracks.forEach(track => track.stop());
  }
};

const captureFrame = async (videoElement) => {
  const canvas = document.createElement('canvas');
  const context = canvas.getContext('2d');

  // Set the canvas dimensions to match the video element
  canvas.width = videoElement.videoWidth;
  canvas.height = videoElement.videoHeight;

  // Draw the current frame from the video element onto the canvas
  context.drawImage(videoElement, 0, 0, canvas.width, canvas.height);

  // Resize the captured frame to a maximum width of 400px
  const maxWidth = 250;
  const scaleFactor = maxWidth / canvas.width;
  const resizedWidth = maxWidth;
  const resizedHeight = canvas.height * scaleFactor;

  canvas.width = resizedWidth;
  canvas.height = resizedHeight;

  // Draw the resized frame onto the canvas
  context.drawImage(videoElement, 0, 0, resizedWidth, resizedHeight);

  // Get the data URL of the resized captured frame in base64 format
  const frameDataUrl = canvas.toDataURL('image/jpeg', 1.0);

  // Do something with the captured frame data
  console.log('Captured frame:', frameDataUrl);

  frame.value = frameDataUrl;

  const response = await openai.chat.completions.create({
    model: "gpt-4-vision-preview",
    max_tokens: 500,
    messages: [
      {
        role: "system",
        content: systemPrompt.value
      },
      {
        role: "user",
        content: [
          { type: "text", text: "Describe this image" },
          {
            type: "image_url",
            image_url: {
              "url": `${frameDataUrl}`,
            },
          },
        ],
      },
    ],
  });
  console.log(response.choices[0].message.content);
  narrationText.value = response.choices[0].message.content;
  makeTTS();

  isRecording.value = false;
  if (isRecording.value) {
    //captureFrame(videoElement);
  }
};

const makeTTS = async () => {
  const requestQueue = []
  const mediaSource = new MediaSource()
  // Set up the MediaSource as the audio element's source
	audioRef.value.src = URL.createObjectURL(mediaSource)

// Start playing the audio element immediately
audioRef.value.play()

mediaSource.addEventListener('sourceopen', () => {


  const sourceBuffer = mediaSource.addSourceBuffer('audio/mpeg') // Adjust the MIME type accordingly

  let isAppending = false
  let appendQueue = []

  function processAppendQueue() {
    if (!isAppending && appendQueue.length > 0) {
      isAppending = true
      const chunk = appendQueue.shift()
      chunk && sourceBuffer.appendBuffer(chunk)
    }
  }

  sourceBuffer.addEventListener('updateend', () => {
    isAppending = false
    processAppendQueue()
  })

  function appendChunk(chunk) {
    appendQueue.push(chunk)
    processAppendQueue()

    while (mediaSource.duration - mediaSource.currentTime > 60) {
      const removeEnd = mediaSource.currentTime - 60
      sourceBuffer.remove(0, removeEnd)
    }
  }

  async function fetchAndAppendChunks() {
    try {
      // Check if the maximum concurrent requests limit is reached
      if (requestQueue.length >= 3) {
        // Queue the request for later execution
        return new Promise((resolve) => {
          requestQueue.push(resolve)
        })
      }

      // Fetch a chunk of audio data
      const response = await fetch(`https://api.elevenlabs.io/v1/text-to-speech/${voiceId.value}/stream?optimize_streaming_latency=0`, {
        method: 'POST',
        headers: {
          Accept: 'audio/mpeg',
          'Content-Type': 'application/json',
          'xi-api-key': elevenlabsApiKey.value // Put in your own API key
        },
        body: JSON.stringify({
    "text": narrationText.value,
    "model_id": "eleven_turbo_v2",
    "voice_settings": {
      "stability": 0,
      "similarity_boost": 0,
      "style": 0,
    }
  })
      })

      if (!response.body) {
        // Streaming is not supported in this response, handle appropriately
        console.error('Streaming not supported by the server')
        return
      }

      const reader = response.body.getReader()

      while (true) {
        const { done, value } = await reader.read()

        if (done) {
          break // No more data to read
        }

        // Append the received chunk to the buffer
        appendChunk(value.buffer)
      }
    } catch (error) {
      console.error('Error fetching and appending chunks:', error)
    } finally {
      // Remove the request from the queue
      const nextRequest = requestQueue.shift()
      if (nextRequest) {
        nextRequest()
      }
    }
  }

  // Call the function to start fetching and appending audio chunks
  fetchAndAppendChunks()
})
}

const makeTextToSpeechRequest = async () => {
  const url = 'https://api.elevenlabs.io/v1/text-to-speech/hfvKg8HifsF5B2g0e0QB/stream?optimize_streaming_latency=0&output_format=mp3_44100_128';
  const headers = {
    'accept': '*/*',
    'Content-Type': 'application/json',
    'xi-api-key': elevenlabsApiKey.value,
  };
  const data = {
    "text": narrationText.value,
    "model_id": "eleven_monolingual_v1",
    "voice_settings": {
      "stability": 0,
      "similarity_boost": 0,
      "style": 0,
      "use_speaker_boost": true
    }
  };

  try {
    console.log('t')
    const mediaSource = new MediaSource();
    audioRef.value.src = URL.createObjectURL(mediaSource);
    console.log('ref')

    const sourceBuffer = mediaSource.addSourceBuffer('audio/mpeg');
    console.log('res')

    const response = await fetch(url, {
      method: 'POST',
      headers,
      body: JSON.stringify(data)
    });
    console.log('req')

    const reader = response.body.getReader();

    const pump = async () => {
      while (true) {
        console.log('in')

        const { done, value } = await reader.read();
        if (done) {
          mediaSource.endOfStream();
          break;
        }
        sourceBuffer.appendBuffer(value);
      }
    };
    console.log('pump')

    pump();
  } catch (error) {
    console.error('Error:', error);
  }
};
</script>