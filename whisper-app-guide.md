# How to build a real-time audio transcription app with Whisper and Together AI

*August 2025 ・ By Riccardo Giorato*

Whisper is an audio transcription app that converts speech to text almost instantly. It's built using Together AI's Whisper model and supports both live recording and file uploads.

![Whisper example](./public/og.jpg)

In this post, you'll learn how to build the core parts of Whisper. The app is open-source and built with Next.js, tRPC for type safety, and Together AI's API, but the concepts can be applied to any language or framework.

## Building the audio recording interface

Whisper's core interaction is a recording modal where users can capture audio directly in the browser:

```tsx
function RecordingModal({ onClose }: { onClose: () => void }) {
  const { recording, audioBlob, startRecording, stopRecording } = useAudioRecording();

  const handleRecordingToggle = async () => {
    if (recording) {
      stopRecording();
    } else {
      await startRecording();
    }
  };

  // Auto-process when we get an audio blob
  useEffect(() => {
    if (audioBlob) {
      handleSaveRecording();
    }
  }, [audioBlob]);

  return (
    <Dialog open onOpenChange={onClose}>
      <DialogContent>
        <Button onClick={handleRecordingToggle}>
          {recording ? "Stop Recording" : "Start Recording"}
        </Button>
      </DialogContent>
    </Dialog>
  );
}
```

The magic happens in our custom `useAudioRecording` hook, which handles all the browser audio recording logic.

## Recording audio in the browser

To capture audio, we use the MediaRecorder API with a simple hook:

```tsx
function useAudioRecording() {
  const [recording, setRecording] = useState(false);
  const [audioBlob, setAudioBlob] = useState<Blob | null>(null);
  
  const mediaRecorderRef = useRef<MediaRecorder | null>(null);
  const chunksRef = useRef<Blob[]>([]);

  const startRecording = async () => {
    try {
      // Request microphone access
      const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
      
      // Create MediaRecorder
      const mediaRecorder = new MediaRecorder(stream);
      mediaRecorderRef.current = mediaRecorder;
      chunksRef.current = [];
      
      // Collect audio data
      mediaRecorder.ondataavailable = (e) => {
        chunksRef.current.push(e.data);
      };
      
      // Create blob when recording stops
      mediaRecorder.onstop = () => {
        const blob = new Blob(chunksRef.current, { type: "audio/webm" });
        setAudioBlob(blob);
        // Stop all tracks to release microphone
        stream.getTracks().forEach(track => track.stop());
      };
      
      mediaRecorder.start();
      setRecording(true);
    } catch (err) {
      console.error("Microphone access denied:", err);
    }
  };

  const stopRecording = () => {
    if (mediaRecorderRef.current && recording) {
      mediaRecorderRef.current.stop();
      setRecording(false);
    }
  };

  return { recording, audioBlob, startRecording, stopRecording };
}
```

This simplified version focuses on the core functionality: start recording, stop recording, and get the audio blob.

## Uploading and transcribing audio

Once we have our audio blob (from recording) or file (from upload), we need to send it to Together AI's Whisper model. We use S3 for temporary storage and tRPC for type-safe API calls:

```tsx
const handleSaveRecording = async () => {
  if (!audioBlob) return;
  
  try {
    // Upload to S3
    const file = new File([audioBlob], `recording-${Date.now()}.webm`, {
      type: "audio/webm",
    });
    const { url } = await uploadToS3(file);
    
    // Call our tRPC endpoint
    const { id } = await transcribeMutation.mutateAsync({
      audioUrl: url,
      language: selectedLanguage,
      durationSeconds: duration,
    });
    
    // Navigate to transcription page
    router.push(`/whispers/${id}`);
  } catch (err) {
    toast.error("Failed to transcribe audio. Please try again.");
  }
};
```

## Creating the transcription API with tRPC

Our backend uses tRPC to provide end-to-end type safety. Here's our transcription endpoint:

```tsx
export const whisperRouter = t.router({
  transcribeFromS3: protectedProcedure
    .input(
      z.object({
        audioUrl: z.string(),
        language: z.string().optional(),
        durationSeconds: z.number().min(1),
      })
    )
    .mutation(async ({ input, ctx }) => {
      // Call Together AI's Whisper model
      const res = await togetherBaseClientWithKey(
        ctx.togetherApiKey
      ).audio.transcriptions.create({
        file: input.audioUrl,
        model: "openai/whisper-large-v3",
        language: input.language || "en",
      });

      const transcription = res.text as string;

      // Generate a title using LLM
      const { text: title } = await generateText({
        prompt: `Generate a title for the following transcription with max of 10 words: ${transcription}`,
        model: togetherVercelAiClient(ctx.togetherApiKey)(
          "meta-llama/Llama-3.3-70B-Instruct-Turbo"
        ),
        maxTokens: 10,
      });

      // Save to database
      const whisperId = uuidv4();
      await prisma.whisper.create({
        data: {
          id: whisperId,
          title: title.slice(0, 80),
          userId: ctx.auth.userId,
          fullTranscription: transcription,
          audioTracks: {
            create: [{
              fileUrl: input.audioUrl,
              partialTranscription: transcription,
              language: input.language,
            }],
          },
        },
      });

      return { id: whisperId };
    }),
});
```

The beauty of tRPC is that our frontend gets full TypeScript intellisense and type checking for this API call.

## Supporting file uploads

For users who want to upload existing audio files, we use react-dropzone and next-s3-upload.

Next-s3-upload handles the S3 upload in the backend and fully integrates with Next.js API routes in a simple 5 minute setup you can read more here: https://next-s3-upload.codingvalue.com/
:

```tsx
import Dropzone from "react-dropzone";
import { useS3Upload } from "next-s3-upload";

function UploadModal({ onClose }: { onClose: () => void }) {
  const { uploadToS3 } = useS3Upload();

  const handleDrop = useCallback(async (acceptedFiles: File[]) => {
    const file = acceptedFiles[0];
    if (!file) return;
    
    try {
      // Get audio duration and upload in parallel
      const [duration, { url }] = await Promise.all([
        getDuration(file),
        uploadToS3(file),
      ]);
      
      // Transcribe using the same endpoint
      const { id } = await transcribeMutation.mutateAsync({
        audioUrl: url,
        language,
        durationSeconds: Math.round(duration),
      });
      
      router.push(`/whispers/${id}`);
    } catch (err) {
      toast.error("Failed to transcribe audio. Please try again.");
    }
  }, []);

  return (
    <Dropzone
      accept={{
        "audio/mpeg3": [".mp3"],
        "audio/wav": [".wav"],
        "audio/mp4": [".m4a"],
      }}
      onDrop={handleDrop}
    >
      {({ getRootProps, getInputProps }) => (
        <div {...getRootProps()}>
          <input {...getInputProps()} />
          <p>Drop audio files here or click to upload</p>
        </div>
      )}
    </Dropzone>
  );
}
```

## Adding audio transformations

Once we have a transcription, users can transform it using LLMs. We support summarization, extraction, and custom transformations:

```tsx
const transformText = async (prompt: string, transcription: string) => {
  const { text } = await generateText({
    prompt: `${prompt}\n\nTranscription: ${transcription}`,
    model: togetherVercelAiClient(apiKey)(
      "meta-llama/Llama-3.3-70B-Instruct-Turbo"
    ),
  });
  
  return text;
};
```

## Type safety with tRPC

One of the key benefits of using tRPC is the end-to-end type safety. When we call our API from the frontend:

```tsx
const transcribeMutation = useMutation(
  trpc.whisper.transcribeFromS3.mutationOptions()
);

// TypeScript knows the exact shape of the input and output
const result = await transcribeMutation.mutateAsync({
  audioUrl: "...",
  language: "en", // TypeScript validates this
  durationSeconds: 120,
});

// result.id is properly typed
router.push(`/whispers/${result.id}`);
```

This eliminates runtime errors and provides excellent developer experience with autocomplete and type checking.

## Going beyond basic transcription

Whisper is open-source, so check out the [full code](https://github.com/nutlope/whisper-app) to learn more and get inspired to build your own audio transcription apps.

When you're ready to start transcribing audio in your own apps, sign up for [Together AI](https://togetherai.link) today and make your first API call in minutes!