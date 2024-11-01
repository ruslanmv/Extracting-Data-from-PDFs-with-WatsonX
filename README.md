## Extracting Data from PDFs with Llama on Watsonx.ai: A Complete Example

In this blog post, we’ll walk through building a pipeline to extract precise data from PDFs using IBM’s Watsonx.ai with Meta’s `llama-3-70b-instruct` model. A sample PDF and expected output are included to help you test and understand the workflow.

### Why Use Llama on Watsonx.ai?

Watsonx.ai provides a platform for deploying large language models (LLMs) with greater control over model parameters and powerful capabilities for question-answering, summarization, and other NLP tasks. By using Meta’s `llama-3-70b-instruct` model on Watsonx.ai, you gain:
- **Advanced Language Understanding**: Llama-3 is optimized for dialogue use cases, including extraction and summarization tasks.
- **Robust Infrastructure**: Watsonx.ai supports seamless integration with IBM’s cloud infrastructure for scalable processing.
- **Data Privacy**: Watsonx.ai offers secure data processing within IBM’s cloud environment.

### Our Goal

To build a Python-based pipeline that can:
1. **Preprocess PDFs**: Convert PDFs into a format suitable for Llama-3 on Watsonx.ai.
2. **Use the Llama Model on Watsonx.ai**: Send the processed data to Llama-3 for extracting information.
3. **Extract Key Information**: Identify and extract company names and activities from the content.

---

### Sample PDF (`example.pdf`)

To test the pipeline, use the following text in a sample PDF named `example.pdf`:

```
Acme Corp is a leading technology company specializing in artificial intelligence and machine learning solutions. Based in San Francisco, Acme Corp develops cutting-edge software for data analysis, natural language processing, and computer vision.

Beta Industries is a global manufacturing company headquartered in New York City. They are involved in the production of sustainable materials and renewable energy technologies. Beta Industries is committed to environmental responsibility and innovation.
```

---

### Step 1: Setting up Your Environment

#### Install Dependencies

```bash
pip install -U ibm-watsonx-ai PyPDF2
```

#### Define Watsonx.ai Credentials

Define your Watsonx.ai credentials to authenticate and access the Llama model. 

```python
import getpass
from ibm_watsonx_ai import Credentials

credentials = Credentials(
    url="https://us-south.ml.cloud.ibm.com",
    api_key=getpass.getpass("Please enter your WML api key (hit enter): "),
)
```

#### Define Project ID

Specify the project ID required by Watsonx.ai to provide context for the model call.

```python
import os

try:
    project_id = os.environ["PROJECT_ID"]
except KeyError:
    project_id = input("Please enter your project_id (hit enter): ")
```

---

### Step 2: Selecting the Llama Model on Watsonx.ai

Define the `model_id` to specify Meta’s `llama-3-70b-instruct` model.

```python
model_id = 'meta-llama/llama-3-70b-instruct'
```

Define the model parameters, such as the decoding method and maximum number of tokens.

```python
from ibm_watsonx_ai.metanames import GenTextParamsMetaNames as GenParams

parameters = {
    GenParams.DECODING_METHOD: "greedy",
    GenParams.MAX_NEW_TOKENS: 100,
    GenParams.STOP_SEQUENCES: ["\n\n"]
}
```

Initialize the model with the specified parameters.

```python
from ibm_watsonx_ai.foundation_models import ModelInference

model = ModelInference(
    model_id=model_id, 
    params=parameters, 
    credentials=credentials,
    project_id=project_id
)
```

---

### Step 3: Preprocessing PDFs

Prepare PDFs by converting them into clean, structured text data for the model.

```python
import PyPDF2

def preprocess_pdf(pdf_path):
    """
    Extracts text from a PDF and performs basic cleaning.

    Args:
        pdf_path: Path to the PDF file.

    Returns:
        cleaned_text: A string containing the extracted and cleaned text.
    """
    text = ""
    with open(pdf_path, 'rb') as pdf_file:
        pdf_reader = PyPDF2.PdfReader(pdf_file)
        for page in pdf_reader.pages:
            text += page.extract_text()
    cleaned_text = " ".join(text.split())  # Remove extra whitespace
    return cleaned_text
```

---

### Step 4: Interacting with the Llama Model on Watsonx.ai

Define a function to send the extracted text to Llama for analysis.

```python
def get_llama_response(text, prompt_template):
    """
    Sends a request to the Llama model on Watsonx.ai and returns the response.

    Args:
        text: The text extracted from the PDF.
        prompt_template: A template for the prompt, including placeholders for the text.

    Returns:
        response_text: The text generated by Llama.
    """
    prompt = prompt_template.format(text=text)
    response = model.generate_text(prompt)
    return response
```

---

### Step 5: Extracting Information

Craft the right prompt to instruct Llama to extract specific details such as company names and activities.

```python
import json

def extract_information(llama_response):
    """
    Extracts company names and activities from the Llama model response.

    Args:
        llama_response: The text generated by Llama.

    Returns:
        extracted_data: A dictionary containing the extracted information.
    """
    prompt_template = """
    Identify the companies mentioned in this text and their main activities:
    {text}

    Provide your answer in the following JSON format:
    {{"companies": [
        {{"name": "company name 1", "activity": "main activity of company 1"}},
        {{"name": "company name 2", "activity": "main activity of company 2"}},
        ...
    ]}}
    """

    response = get_llama_response(llama_response, prompt_template)
    try:
        data = json.loads(response)
        return data
    except json.JSONDecodeError:
        print("Error: Invalid JSON response from Llama.")
        return None
```

---

### Step 6: Putting it All Together

Combine these functions into a complete pipeline to process the PDF and extract relevant information.

```python
def process_pdf(pdf_path):
    """
    Processes a PDF to extract company names and activities.

    Args:
        pdf_path: Path to the PDF file.

    Returns:
        extracted_data: A dictionary containing the extracted information.
    """
    text = preprocess_pdf(pdf_path)
    llama_response = get_llama_response(text, prompt_template)
    extracted_data = extract_information(llama_response)
    return extracted_data

# Example usage
pdf_path = "example.pdf"  # Replace with your sample PDF path
extracted_data = process_pdf(pdf_path)
print(extracted_data)
```

---

### Expected Output

After running the code, you should see an output similar to the following JSON format:

```json
{
  "companies": [
    {
      "name": "Acme Corp",
      "activity": "technology company specializing in artificial intelligence and machine learning solutions"
    },
    {
      "name": "Beta Industries",
      "activity": "global manufacturing company involved in the production of sustainable materials and renewable energy technologies"
    }
  ]
}
```

---

### Running the Example

1. **Save the Code**: Save the code above as a Python file (e.g., `pdf_extractor_llama_watsonx.py`).
2. **Create `example.pdf`**: Create a PDF file named `example.pdf` with the sample text provided.
3. **Set Up Watsonx.ai**: Ensure your IBM Cloud Project and Watsonx.ai settings are configured correctly.
4. **Run the Script**: Execute the script from your terminal: `python pdf_extractor_llama_watsonx.py`

---

### Important Notes

- **Fine-tuning**: Fine-tune the Llama model on Watsonx.ai for similar PDF datasets to improve extraction accuracy.
- **Prompt Engineering**: Experiment with different prompts and response structures to optimize the extraction process.
- **Error Handling**: Implement robust error handling to manage invalid PDFs or unexpected responses from the model.
- **Resource Management**: Monitor usage on Watsonx.ai to ensure efficient handling of larger datasets.

### Conclusion

This guide provides a comprehensive approach to building a PDF data extraction pipeline using IBM’s Watsonx.ai and Meta’s `llama-3-70b-instruct`. By leveraging Watsonx.ai’s infrastructure, this pipeline offers a flexible, scalable solution for extracting structured data from PDFs, making it ideal for document-heavy workflows that require data privacy and secure processing. With additional fine-tuning, this setup can be adapted for broader use cases across various industries.