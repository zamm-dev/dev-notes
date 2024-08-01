# Setting up Llama

## Setup Ollama

Download Ollama from [their website](https://ollama.com). Then run

```bash
$ ollama run llama3
pulling manifest 
pulling 6a0746a1ec1a... 100% ▕████████████████▏ 4.7 GB                         
pulling 4fa551d4f938... 100% ▕████████████████▏  12 KB                         
pulling 8ab4849b038c... 100% ▕████████████████▏  254 B                         
pulling 577073ffcc6c... 100% ▕████████████████▏  110 B                         
pulling 3f8eb4da87fa... 100% ▕████████████████▏  485 B                         
verifying sha256 digest 
writing manifest 
removing any unused layers 
success 
>>> Send a message (/? for help)
```

While that runs, we can even use the instructions [here](https://llama.meta.com/docs/llama-everywhere/running-meta-llama-on-mac/) and make an HTTP call to it:

```bash
$ curl http://localhost:11434/api/chat -d '{
  "model": "llama3",
  "messages": [
    {
      "role": "user",
      "content": "who wrote the book godfather?"
    }
  ],
  "stream": false
}'
{"model":"llama3","created_at":"2024-07-29T13:15:18.083511Z","message":{"role":"assistant","content":"The novel \"The Godfather\" was written by Mario Puzo. The book was published in 1969 and has since become a classic of American literature.\n\nMario Puzo was an American author, screenwriter, and playwright who was born in 1920 and passed away in 1999. He is best known for writing the novel \"The Godfather,\" which was adapted into a successful film directed by Francis Ford Coppola in 1972.\n\nPuzo's novel tells the story of the Corleone crime family, led by Don Vito Corleone (played by Marlon Brando in the film), and explores themes of loyalty, power, and betrayal. The book has been widely praised for its vivid portrayal of Italian-American culture and its exploration of the American Dream.\n\nIt's worth noting that Puzo collaborated with Francis Ford Coppola on the screenplay for the 1972 film adaptation of \"The Godfather,\" which won several Academy Awards and is considered one of the greatest films of all time."},"done_reason":"stop","done":true,"total_duration":11491375833,"load_duration":28042041,"prompt_eval_count":17,"prompt_eval_duration":421645000,"eval_count":209,"eval_duration":11040562000}
```
