# QA Bot Interface with Gradio on Google Colab
# Front end
# First, let's install the required libraries
!pip install -q transformers sentence-transformers faiss-cpu gradio PyPDF2

# Import necessary libraries
import gradio as gr
import PyPDF2
import io
from transformers import pipeline
from sentence_transformers import SentenceTransformer
import faiss
import numpy as np

# Load models
qa_model = pipeline("question-answering", model="distilbert-base-cased-distilled-squad")
embedding_model = SentenceTransformer('all-MiniLM-L6-v2')

# Initialize global variables
document_text = ""
embeddings = None
faiss_index = None
sentences = []

def process_pdf(pdf_file):
    global document_text, embeddings, faiss_index, sentences

    pdf_reader = PyPDF2.PdfReader(pdf_file)
    document_text = ""
    for page in pdf_reader.pages:
        document_text += page.extract_text()

    sentences = document_text.split('. ')
    embeddings = embedding_model.encode(sentences)
    faiss_index = faiss.IndexFlatL2(embeddings.shape[1])
    faiss_index.add(embeddings)

    return "Document processed successfully!"

def get_relevant_context(query, k=3):
    query_vector = embedding_model.encode([query])
    _, I = faiss_index.search(query_vector, k)
    relevant_sentences = [sentences[i] for i in I[0]]
    return ". ".join(relevant_sentences)

def answer_question(query):
    if not document_text:
        return "Please upload a document first.", ""

    relevant_context = get_relevant_context(query)
    answer = qa_model(question=query, context=relevant_context)

    return answer['answer'], relevant_context

# Define the Gradio interface
with gr.Blocks() as demo:
    gr.Markdown("# Interactive QA Bot")

    with gr.Row():
        pdf_input = gr.File(label="/content/drive/MyDrive/pdfs/A_Machine_Learning_Assisted_Method_of_Coverage_and_Capacity_Optimization_CCO_in_4G_LTE_Self_Organizing_Networks_SON.pdf")
        process_button = gr.Button("Process Document")

    output_text = gr.Textbox(label="Processing Status")

    query_input = gr.Textbox(label="Ask a question about the document")
    submit_button = gr.Button("Submit")

    answer_output = gr.Textbox(label="Answer")
    context_output = gr.Textbox(label="Relevant Context")

    process_button.click(process_pdf, inputs=pdf_input, outputs=output_text)
    submit_button.click(answer_question, inputs=query_input, outputs=[answer_output, context_output])

    gr.Markdown("## How to use this QA Bot:")
    gr.Markdown("""
    1. Upload a PDF document using the file uploader above.
    2. Click the "Process Document" button and wait for the success message.
    3. Type your question in the text input field.
    4. Click the "Submit" button to get an answer based on the document's content.
    5. You can see the relevant context used to generate the answer below the response.
    """)

    gr.Markdown("## Example Interactions:")
    gr.Markdown("""
    - Q: "What is the main topic of the document?"
    - Q: "Can you summarize the key points in the first section?"
    - Q: "Are there any statistics or numbers mentioned in the document?"

    Note: The quality of answers depends on the content of the uploaded document and the specificity of your questions.
    """)

# Launch the interface
demo.launch(debug=True)
# Backend
import PyPDF2
from transformers import pipeline
from sentence_transformers import SentenceTransformer
import faiss
import numpy as np

# Load models (place outside functions)
qa_model = pipeline("question-answering", model="distilbert-base-cased-distilled-squad")
embedding_model = SentenceTransformer('all-MiniLM-L6-v2')

# Function to process the PDF and generate embeddings
def process_pdf(pdf_file):
    pdf_reader = PyPDF2.PdfReader(pdf_file)
    document_text = ""
    for page in pdf_reader.pages:
        document_text += page.extract_text()

    sentences = document_text.split('. ')  # Split into sentences
    embeddings = embedding_model.encode(sentences)  # Generate sentence embeddings

    # Create Faiss index for efficient retrieval (optional)
    faiss_index = faiss.IndexFlatL2(embeddings.shape[1])
    faiss_index.add(embeddings)

    return sentences, embeddings, faiss_index  # Return extracted data

# Function to retrieve relevant context based on a query
def get_relevant_context(query, sentences, faiss_index=None, k=3):
    if faiss_index:
        query_vector = embedding_model.encode([query])
        _, I = faiss_index.search(query_vector, k)
        relevant_sentences = [sentences[i] for i in I[0]]
    else:
        # If no Faiss index, use a simpler approach (e.g., keyword matching)
        relevant_sentences = [s for s in sentences if query.lower() in s.lower()]

    return ". ".join(relevant_sentences)

# Function to answer the question using the QA model and relevant context
def answer_question(query, context):
    answer = qa_model(question=query, context=context)
    return answer['answer'], context
