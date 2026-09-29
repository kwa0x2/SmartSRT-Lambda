# SmartSRT Lambda

The transcription step of [SmartSRT](https://smartsrt.com), a subtitle generation service. This AWS Lambda function takes an uploaded audio or video file from S3, transcribes it with the OpenAI Whisper API and writes an `.srt` subtitle file back to S3.

Users never call it directly. The [SmartSRT backend](https://github.com/kwa0x2/SmartSRT-Backend) (Go) queues each upload in RabbitMQ, and its worker pool invokes this function synchronously with `lambda:Invoke`. The response carries the S3 URL of the subtitle file, which the [frontend](https://github.com/kwa0x2/SmartSRT-Frontend) then offers for download.

![Flow](diagram.png)

## How it works

1. Validates the payload (see below).
2. Downloads `files/{user_id}/{file_name}` from the S3 bucket.
3. Sends the file to `POST /v1/audio/transcriptions` with `model=whisper-1`, `response_format=verbose_json` and `timestamp_granularities[]=word`, so every word comes back with a start and an end time.
4. Groups the words into subtitle entries according to the options in the payload.
5. Uploads the result to `srt-files/{user_id}/{file_name}.srt` with `Content-Type: application/x-subrip`.
6. Returns the status code and the URL of the file.

Each step is wrapped in `console.time`, so the CloudWatch log of every invocation shows where the time went.

## Input

```json
{
  "file_name": "example_video.mp4",
  "user_id": "6790a64b87c2387f93afd485",
  "words_per_line": 3,
  "punctuation": true,
  "consider_punctuation": true
}
```

| Field | Type | Description |
|---|---|---|
| `file_name` | string | Name of the uploaded file. Must end in `.mp4`, `.mp3` or `.wav`. |
| `user_id` | string | Owner of the file. Used to build the S3 keys. |
| `words_per_line` | number | Maximum number of words in one subtitle entry, 1 to 5. |
| `punctuation` | boolean | Keep punctuation in the subtitles. |
| `consider_punctuation` | boolean | End an entry at `.`, `!` or `?` even before `words_per_line` is reached. Requires `punctuation: true`. |

### The two punctuation options

Whisper returns the full transcript with punctuation, but the word level timestamps come back as bare words without it. With `punctuation: true` the function walks both lists together and matches each punctuated word of the transcript to its timestamped counterpart (or counterparts, when Whisper splits one word into several), so the subtitles keep commas and full stops and still carry accurate timings. With `punctuation: false` the timestamped words are used as they are.

`consider_punctuation` only makes sense when punctuation is kept, so the function rejects `punctuation: false` combined with `consider_punctuation: true`.

## Output

```json
{
  "status_code": 200,
  "body": {
    "message": "SRT file generated successfully!",
    "srt_url": "https://<bucket>.s3.<region>.amazonaws.com/srt-files/6790a64b87c2387f93afd485/example_video.srt"
  }
}
```

On failure `status_code` is `400` for an invalid payload, the status returned by the Whisper API when that call fails, or `500` for anything else. `body.message` carries the reason and `srt_url` is empty. The function catches all errors and returns them in this shape instead of throwing, and the backend checks `status_code`.

## Limits

- Input formats: MP4, MP3 and WAV.
- The file goes to Whisper in a single request, so it has to stay under the Whisper API's 25 MB upload limit.
- The function timeout is 5 minutes, and the S3 and OpenAI clients use the same timeout.

## Deployment

Set up from the AWS console:

1. Create a Lambda function on a Node.js 18 or newer runtime and upload `index.mjs` as the source. The handler is `index.handler`.
2. Attach the two layers from `layers/`: `aws-sdk-layer.zip` (AWS SDK v2, which Node.js 18+ runtimes no longer bundle) and `axios-layer.zip` (axios and form-data).
3. Set memory to 2048 MB (Lambda scales CPU and network with memory) and the timeout to 5 minutes.
4. Fill in `AWS_REGION`, `BUCKET_NAME` and `OPENAI_API_KEY` in the `CONFIG` block at the top of `index.mjs`.
5. Give the execution role `s3:GetObject` and `s3:PutObject` on the bucket. The caller (the SmartSRT backend) needs `lambda:InvokeFunction` on this function.

## Performance

Timings from a CloudWatch log for a 4 MB video:

| Step | Time |
|---|---|
| S3 download | 0.47 s |
| Whisper API | 3.17 s |
| SRT generation | under 1 ms |
| S3 upload | 65 ms |
| Total | 3.7 s |

Peak memory use was 144 MB.

<details>
<summary>Raw log</summary>

```
START RequestId: da692ecb-b3f1-4a92-8c34-ded39595029a Version: $LATEST
2025-03-19T11:37:19.734Z	da692ecb-b3f1-4a92-8c34-ded39595029a	INFO	s3-fetch: 470.787ms
2025-03-19T11:37:22.908Z	da692ecb-b3f1-4a92-8c34-ded39595029a	INFO	whisper-api: 3.173s
2025-03-19T11:37:22.909Z	da692ecb-b3f1-4a92-8c34-ded39595029a	INFO	srt-generation: 0.909ms
2025-03-19T11:37:22.974Z	da692ecb-b3f1-4a92-8c34-ded39595029a	INFO	s3-upload: 65.06ms
2025-03-19T11:37:22.974Z	da692ecb-b3f1-4a92-8c34-ded39595029a	INFO	total-execution: 3.711s
END RequestId: da692ecb-b3f1-4a92-8c34-ded39595029a
REPORT RequestId: da692ecb-b3f1-4a92-8c34-ded39595029a	Duration: 3717.83 ms	Billed Duration: 3718 ms	Memory Size: 2048 MB	Max Memory Used: 144 MB	Init Duration: 853.54 ms
```

</details>

## Related

- [SmartSRT-Backend](https://github.com/kwa0x2/SmartSRT-Backend): Go API and RabbitMQ consumer that invokes this function
- [SmartSRT-Frontend](https://github.com/kwa0x2/SmartSRT-Frontend): Next.js web client
- [smartsrt.com](https://smartsrt.com): the live service

## License

MIT
