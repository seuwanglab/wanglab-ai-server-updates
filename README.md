# wanglab-ai-server-updates


## 2025-11-20

### New Features
- Significantly optimized response speed for GPT Series.
- Added truncation function to reduce error possibilities.([Technical Details](supplyment/11-20-2025-truncation_filter.md))

### More Details
- Optimized response speed for GPT 4o/4.1/5/5.1 models.
- Introduced smart truncation function to manage context length and prevent errors.  

## 2025-11-19

### New Features
- Added **Online Search** button.
- Added **Context Round Truncation** feature for dialogues.
- Updated latest three models.
- Improved model descriptions for better usability.
- New **Document Retrieval** and **Hybrid Query** functionality.

### More Details
- Added:
    - “Web Search” toggle button for convenient online search.
    - Context message truncation for multi-turn conversations to improve stability and reduce token usage.
    
![Updated Features](img/11-19-2025-func.png)

- Updated Models:
    - Gemini 3 Pro
    - Grok 4 Fast
    - Grok 4.1

- Introduced a new document retrieval pipeline with hybrid query capabilities.  
    - Current pipeline:
        - **OCR**: Mistral OCR
        - **Embedding**: Qwen3-Embedding-8B  
            - Chunk size: 2048  
            - Chunk overlap: 256
        - **Hybrid search reranking**: Qwen3-Reranker-8B ([Technical Details](supplyment/11-19-2025-qwen3-rerank-8b.md))  
            - `top_k`: 10  
            - `top_k_rank`: 5  
            - Score threshold: 0.4

## License

This project is licensed under the Apache License 2.0 - see the [LICENSE](LICENSE) file for details.
        

