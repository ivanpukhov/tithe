/* ChatPage.css */
.chat-page {
max-width: 600px;
margin: auto;
padding: 20px;
}

.api-token-input, .message-input {
width: 100%;
padding: 10px;
margin-bottom: 10px;
border: 1px solid #ccc;
border-radius: 4px;
}

.send-button {
width: 100%;
padding: 10px;
background-color: #007bff;
color: white;
border: none;
border-radius: 4px;
cursor: pointer;
}

.send-button:disabled {
background-color: #ccc;
}

.chat-container {
border: 1px solid #ddd;
padding: 10px;
min-height: 200px;
margin-bottom: 10px;
overflow-y: auto;
height: 500px;
max-height: 500px;
display: flex;
flex-direction: column;
}

.message {
margin: 5px 0;
padding: 5px;
border-radius: 4px;
width: 70%;
}

.message.user {
background-color: #e7f3fe;
text-align: right;
float: right;
align-self: flex-end;

}

.message.bot {
background-color: #f1f1f1;
text-align: left;
}

.partial-response {
color: #888;
font-style: italic;
}

.error-message {
color: red;
margin: 10px 0;
}

.cursor-blink {
animation: blink 1s step-end infinite;
}

@keyframes blink {
50% {
opacity: 0;
}
}


// src/App.js
import React from 'react';
import { BrowserRouter as Router, Routes, Route } from 'react-router-dom';
import Navbar from './components/Navbar';
import HomePage from './pages/HomePage';
import ChatPage from './pages/ChatPage';
import ImageGenerationPage from './pages/ImageGenerationPage';
import ImageRecognitionPage from './pages/ImageRecognitionPage';
import './App.css'
function App() {
return (
<Router>
<Navbar />
<Routes>
<Route path="/" element={<HomePage />} />
<Route path="/chat" element={<ChatPage />} />
<Route path="/image-generation" element={<ImageGenerationPage />} />
<Route path="/image-recognition" element={<ImageRecognitionPage />} />
</Routes>
</Router>
);
}

export default App;

// src/pages/ChatPage.js
import React, { useEffect, useState } from 'react';
import axios from 'axios';
import { useApiToken } from '../contexts/ApiTokenContext';
import ApiTokenInput from "../contexts/ApiTokenInput";

function ChatPage() {
const [inputText, setInputText] = useState('');
const savedMessages = JSON.parse(localStorage.getItem('chatMessages') || '[]');
const [messages, setMessages] = useState(savedMessages);
const [isLoading, setIsLoading] = useState(false);
const [error, setError] = useState('');
const [conversationId, setConversationId] = useState(null);
const { apiToken } = useApiToken();

    useEffect(() => {
        localStorage.setItem('chatMessages', JSON.stringify(messages));
    }, [messages]);


    const handleInputChange = (e) => setInputText(e.target.value);

    const handleKeyDown = (e) => {
        if (e.key === 'Enter' && !isLoading) {
            sendMessage();
        }
    };
    const sendMessage = async () => {
        if (!inputText.trim()) return;
        setError('');
        setIsLoading(true);
        setMessages(messages => [...messages, { sender: 'user', text: inputText }]);
        try {
            const response = await axios.post('http://localhost:3000/api/chat/conversation', {
                prompt: inputText,
                conversation_id: conversationId
            }, {
                headers: { 'Authorization': `Bearer ${apiToken}` }
            });

            setConversationId(response.data.conversation_id);
            fetchBotResponse(response.data.conversation_id);
        } catch (err) {
            handleError(err);
        }
        setInputText('');
    };

    const handleClearMessages = () => {
        setMessages([]); // Очищаем массив сообщений
        localStorage.removeItem('chatMessages'); // Удаляем сообщения из localStorage
    };

    const fetchBotResponse = async (conversationId) => {
        try {
            const response = await axios.get(`http://localhost:3000/api/chat/conversation/${conversationId}`, {
                headers: { 'Authorization': `Bearer ${apiToken}` }
            });

            if (!response.data.is_final) {
                setTimeout(() => fetchBotResponse(conversationId), 1000);
            } else {
                setMessages(messages => [...messages, { sender: 'bot', text: '' }]);
                typeWriterEffect(response.data.response);
            }
        } catch (err) {
            handleError(err);
        } finally {
            setIsLoading(false);
        }

    };

    const typeWriterEffect = (text, index = 0, currentResponse = '') => {
        setIsLoading(true); // Включение анимации пишущей машинки
        if (index < text.length) {
            const updatedText = currentResponse + text.charAt(index);
            setMessages(messages => {
                const newMessages = [...messages];
                if (newMessages.length > 0) {
                    const lastMessageIndex = newMessages.length - 1;
                    newMessages[lastMessageIndex] = { ...newMessages[lastMessageIndex], text: updatedText };
                }
                return newMessages;
            });
            setTimeout(() => typeWriterEffect(text, index + 1, updatedText), Math.random() * (20 - 2) + 2);
        } else {
            setIsLoading(false); // Выключение анимации пишущей машинки
        }
    };


    const handleError = (err) => {
        setIsLoading(false);
        if (!err.response) {
            setError('Network error. Please check your connection.');
        } else {
            switch (err.response.status) {
                case 400:
                    setError('Bad Request: Your message is improperly formatted.');
                    break;
                case 401:
                    setError('Unauthorized: Your API token is invalid. Please enter a new API token.');
                    break;
                case 403:
                    setError('Forbidden: Your quota has been exceeded. Please wait until next month or upgrade your quota.');
                    break;
                case 503:
                    setError('Service Unavailable: The service is currently unavailable. Please try again later.');
                    break;
                default:
                    setError('An unexpected error occurred. Please try again.');
            }
        }
        console.error(err);
    };

    return (
        <div className="chat-page">
            <ApiTokenInput />

            <div className="chat-container">
                {messages.map((msg, index) => (
                    <div key={index} className={`message ${msg.sender}`}>
                        {msg.text}
                    </div>
                ))}
                {isLoading && <div className="loading">Bot is typing...</div>}
            </div>
            <input
                type="text"
                value={inputText}
                onChange={handleInputChange}
                onKeyDown={handleKeyDown}
                placeholder="Type a message..."
                className="message-input"
                // disabled={isLoading}
            />
            <button onClick={sendMessage} className="send-button" disabled={isLoading}>Send</button>
            <button onClick={handleClearMessages} className="clear-button" disabled={isLoading}>Clear Messages</button>

            {error && <div className="error-message">{error}</div>}
        </div>
    );
}

export default ChatPage;

// src/pages/ImageGenerationPage.js
import React, { useState } from 'react';
import axios from 'axios';
import { useApiToken } from '../contexts/ApiTokenContext';
import ApiTokenInput from "../contexts/ApiTokenInput";

function ImageGenerationPage() {
const [textPrompt, setTextPrompt] = useState('');
const [generatedImage, setGeneratedImage] = useState(null);
const [isLoading, setIsLoading] = useState(false);
const [progress, setProgress] = useState(0);
const [error, setError] = useState('');
const { apiToken } = useApiToken();

    const handleTextChange = (e) => {
        setTextPrompt(e.target.value);
    };

    const clearText = () => {
        setTextPrompt('');
        setGeneratedImage(null);
        setProgress(0);
    };

    const generateImage = async () => {
        setIsLoading(true);
        setError('');
        setGeneratedImage(null);
        setProgress(0);

        try {
            const response = await axios.post('/api/imagegeneration/generate', {
                text_prompt: textPrompt
            }, {
                headers: { 'Authorization': `Bearer ${apiToken}` }
            });
            pollImageStatus(response.data.job_id);
        } catch (err) {
            handleError(err);
        }
    };

    const pollImageStatus = async (jobId) => {
        try {
            const response = await axios.get(`/api/imagegeneration/status/${jobId}`, {
                headers: { 'Authorization': `Bearer ${apiToken}` }
            });
            setProgress(response.data.progress);

            if (response.data.status === 'finished') {
                setGeneratedImage(response.data.image_url);
                setIsLoading(false);
            } else {
                setTimeout(() => pollImageStatus(jobId), 2000);
            }
        } catch (err) {
            handleError(err);
        }
    };

    const handleError = (err) => {
        setIsLoading(false);
        setError('Error generating image. Please try again.');
        console.error(err);
    };

    const saveImage = () => {
        const link = document.createElement('a');
        link.href = generatedImage;
        link.download = 'generated-image.png';
        link.click();
    };

    return (
        <div className="container mx-auto p-4">
            <ApiTokenInput />

            <textarea
                className="w-full p-2 border border-gray-300 rounded mb-4"
                value={textPrompt}
                onChange={handleTextChange}
                placeholder="Enter text for image generation"
                disabled={isLoading}
            />
            <div className="flex space-x-4 mb-4">
                <button
                    className="bg-blue-500 text-white p-2 rounded disabled:bg-blue-300"
                    onClick={generateImage}
                    disabled={isLoading || !textPrompt}
                >
                    Generate Image
                </button>
                <button
                    className="bg-gray-500 text-white p-2 rounded disabled:bg-gray-300"
                    onClick={clearText}
                    disabled={isLoading}
                >
                    Clear
                </button>
            </div>
            {isLoading && <div className="mb-4">Generating... {progress}%</div>}
            {generatedImage && (
                <div>
                    <img className="max-w-full h-auto" src={generatedImage} alt="Generated" />
                    <div className="flex space-x-4 mt-4">
                        <button className="bg-green-500 text-white p-2 rounded" onClick={saveImage}>Save Image</button>
                        {/* Кнопки для увеличения и уменьшения изображения */}
                    </div>
                </div>
            )}
            {error && <div className="text-red-500">{error}</div>}
        </div>
    );
}

export default ImageGenerationPage;

// src/pages/ImageRecognitionPage.js
import React, { useState } from 'react';
import axios from 'axios';
import { useApiToken } from '../contexts/ApiTokenContext';
import ApiTokenInput from "../contexts/ApiTokenInput";

function ImageRecognitionPage() {
const [selectedImage, setSelectedImage] = useState(null);
const [recognitionResults, setRecognitionResults] = useState([]);
const [isLoading, setIsLoading] = useState(false);
const [error, setError] = useState('');
const { apiToken } = useApiToken();

    const handleImageChange = (event) => {
        if (event.target.files && event.target.files[0]) {
            setSelectedImage(event.target.files[0]);
        }
    };

    const recognizeImage = async () => {
        if (!selectedImage) {
            setError('Please select an image to recognize.');
            return;
        }

        setIsLoading(true);
        setError('');
        const formData = new FormData();
        formData.append('image', selectedImage);

        try {
            const response = await axios.post('/api/imagerecognition/recognize', formData, {
                headers: {
                    'Authorization': `Bearer ${apiToken}`,
                    'Content-Type': 'multipart/form-data'
                }
            });
            setRecognitionResults(response.data.objects);
        } catch (err) {
            handleError(err);
        } finally {
            setIsLoading(false);
        }
    };

    const handleError = (err) => {
        setIsLoading(false);
        setError('Error recognizing the image. Please try again.');
        console.error(err);
    };

    return (
        <div className="container mx-auto p-4">

            <ApiTokenInput />

            <input
                type="file"
                onChange={handleImageChange}
                className="file-input mb-4"
                accept="image/*"
            />
            <button
                onClick={recognizeImage}
                className="bg-blue-500 text-white p-2 rounded disabled:bg-blue-300"
                disabled={isLoading || !selectedImage}
            >
                Recognize Image
            </button>

            {isLoading && <div className="mt-4">Processing image...</div>}
            {recognitionResults.length > 0 && (
                <ul className="list-disc list-inside mt-4">
                    {recognitionResults.map((result, index) => (
                        <li key={index}>{result.name}</li>
                    ))}
                </ul>
            )}
            {error && <div className="text-red-500 mt-4">{error}</div>}
        </div>
    );
}

export default ImageRecognitionPage;

// src/pages/HomePage.js
import React from 'react';
import { Link } from 'react-router-dom';

function HomePage() {
return (
<div className="container mx-auto p-4">
<h1 className="text-xl font-bold text-center mb-4">Welcome to AI Services Hub</h1>
<p className="text-center mb-8">Explore various AI powered services like chat bots, image generation, and image recognition.</p>
<div className="grid grid-cols-1 md:grid-cols-3 gap-4">
<div className="bg-white shadow-lg rounded-lg p-6">
<h2 className="font-bold mb-2">Chat Bot</h2>
<p>Interact with our advanced AI chat bot.</p>
<Link to="/chat" className="text-blue-500">Go to Chat →</Link>
</div>
<div className="bg-white shadow-lg rounded-lg p-6">
<h2 className="font-bold mb-2">Image Generation</h2>
<p>Create images from textual descriptions.</p>
<Link to="/image-generation" className="text-blue-500">Create Images →</Link>
</div>
<div className="bg-white shadow-lg rounded-lg p-6">
<h2 className="font-bold mb-2">Image Recognition</h2>
<p>Upload images and let AI recognize objects.</p>
<Link to="/image-recognition" className="text-blue-500">Recognize Images →</Link>
</div>
</div>
{/* Add more content or footer here */}
</div>
);
}

export default HomePage;

// src/components/ApiTokenInput.js
import React from 'react';
import { useApiToken } from '../contexts/ApiTokenContext';

function ApiTokenInput() {
const { apiToken, updateToken } = useApiToken();

    return (
        <input
            type="text"
            placeholder="Enter API Token"
            value={apiToken}
            onChange={(e) => updateToken(e.target.value)}
            className="border p-2 rounded mb-4 w-full"
        />
    );
}

export default ApiTokenInput;

// src/contexts/ApiTokenContext.js
import React, { createContext, useState, useContext } from 'react';

const ApiTokenContext = createContext();

export const useApiToken = () => useContext(ApiTokenContext);

export const ApiTokenProvider = ({ children }) => {
const [apiToken, setApiToken] = useState(localStorage.getItem('apiToken') || '');

    const updateToken = (token) => {
        localStorage.setItem('apiToken', token);
        setApiToken(token);
    };

    return (
        <ApiTokenContext.Provider value={{ apiToken, updateToken }}>
            {children}
        </ApiTokenContext.Provider>
    );
};

// src/components/Navbar.js
import React from 'react';
import { Link } from 'react-router-dom';

function Navbar() {
return (
<nav className="bg-gray-800 p-4 text-white">
<ul className="flex space-x-4">
<li>
<Link to="/">Home</Link>
</li>
<li>
<Link to="/chat">Chat</Link>
</li>
<li>
<Link to="/image-generation">Image Generation</Link>
</li>
<li>
<Link to="/image-recognition">Image Recognition</Link>
</li>
</ul>
</nav>
);
}

export default Navbar;

import React from 'react';
import ReactDOM from 'react-dom/client';
import './index.css';
import App from './App';
import reportWebVitals from './reportWebVitals';
import {ApiTokenProvider} from "./contexts/ApiTokenContext";

const root = ReactDOM.createRoot(document.getElementById('root'));
root.render(
<React.StrictMode>
<ApiTokenProvider>

            <App/>
        </ApiTokenProvider>
    </React.StrictMode>
);

// If you want to start measuring performance in your app, pass a function
// to log results (for example: reportWebVitals(console.log))
// or send to an analytics endpoint. Learn more: https://bit.ly/CRA-vitals
reportWebVitals();
