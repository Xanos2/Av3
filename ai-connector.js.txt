// AI API Connector
// This implementation uses Fetch API to connect to OpenAI's API

class AIConnector {
  constructor(apiKey, baseUrl = 'https://api.openai.com/v1') {
    this.apiKey = apiKey;
    this.baseUrl = baseUrl;
    this.defaultHeaders = {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${this.apiKey}`
    };
  }

  /**
   * Send a text completion request to the AI API
   * @param {string} prompt - The text prompt to send to the API
   * @param {Object} options - Additional options for the request
   * @returns {Promise<Object>} - The API response
   */
  async getCompletion(prompt, options = {}) {
    const endpoint = '/chat/completions';
    const defaultParams = {
      model: "gpt-3.5-turbo",
      messages: [
        {
          role: "user",
          content: prompt
        }
      ],
      temperature: 0.7,
      max_tokens: 150
    };

    const params = { ...defaultParams, ...options };
    
    try {
      const response = await fetch(`${this.baseUrl}${endpoint}`, {
        method: 'POST',
        headers: this.defaultHeaders,
        body: JSON.stringify(params)
      });

      if (!response.ok) {
        const errorData = await response.json();
        throw new Error(`API error: ${errorData.error?.message || response.statusText}`);
      }

      return await response.json();
    } catch (error) {
      console.error('Error connecting to AI API:', error);
      throw error;
    }
  }

  /**
   * Generate an image from a text prompt
   * @param {string} prompt - The text description of the desired image
   * @param {Object} options - Additional options for the request
   * @returns {Promise<Object>} - The API response with image URL(s)
   */
  async generateImage(prompt, options = {}) {
    const endpoint = '/images/generations';
    const defaultParams = {
      prompt,
      n: 1,
      size: "1024x1024"
    };

    const params = { ...defaultParams, ...options };
    
    try {
      const response = await fetch(`${this.baseUrl}${endpoint}`, {
        method: 'POST',
        headers: this.defaultHeaders,
        body: JSON.stringify(params)
      });

      if (!response.ok) {
        const errorData = await response.json();
        throw new Error(`API error: ${errorData.error?.message || response.statusText}`);
      }

      return await response.json();
    } catch (error) {
      console.error('Error connecting to AI API for image generation:', error);
      throw error;
    }
  }

  /**
   * Stream responses from the API
   * @param {string} prompt - The text prompt to send
   * @param {Function} onChunk - Callback function for each chunk of streamed data
   * @param {Object} options - Additional options for the request
   */
  async streamCompletion(prompt, onChunk, options = {}) {
    const endpoint = '/chat/completions';
    const defaultParams = {
      model: "gpt-3.5-turbo",
      messages: [
        {
          role: "user",
          content: prompt
        }
      ],
      temperature: 0.7,
      max_tokens: 150,
      stream: true
    };

    const params = { ...defaultParams, ...options };
    
    try {
      const response = await fetch(`${this.baseUrl}${endpoint}`, {
        method: 'POST',
        headers: this.defaultHeaders,
        body: JSON.stringify(params)
      });

      if (!response.ok) {
        const errorData = await response.json();
        throw new Error(`API error: ${errorData.error?.message || response.statusText}`);
      }

      const reader = response.body.getReader();
      const decoder = new TextDecoder("utf-8");
      
      while (true) {
        const { done, value } = await reader.read();
        if (done) break;
        
        const chunk = decoder.decode(value);
        const lines = chunk.split('\n').filter(line => line.trim() !== '');
        
        for (const line of lines) {
          if (line.startsWith('data: ')) {
            const data = line.slice(6);
            if (data === '[DONE]') break;
            
            try {
              const parsedData = JSON.parse(data);
              onChunk(parsedData);
            } catch (e) {
              console.warn('Error parsing stream data:', e);
            }
          }
        }
      }
    } catch (error) {
      console.error('Error connecting to AI API stream:', error);
      throw error;
    }
  }
}

// Example usage:
async function example() {
  try {
    // Initialize the connector with your API key
    const connector = new AIConnector('your-api-key-here');
    
    // Example 1: Get a completion
    const completionResponse = await connector.getCompletion(
      "Explain quantum computing in simple terms"
    );
    console.log('Completion:', completionResponse.choices[0].message.content);
    
    // Example 2: Generate an image
    const imageResponse = await connector.generateImage(
      "A serene landscape with mountains and a lake at sunset"
    );
    console.log('Generated image URL:', imageResponse.data[0].url);
    
    // Example 3: Stream a completion
    await connector.streamCompletion(
      "Write a short story about a robot learning to paint",
      (chunk) => {
        if (chunk.choices && chunk.choices[0].delta.content) {
          process.stdout.write(chunk.choices[0].delta.content);
        }
      }
    );
    
  } catch (error) {
    console.error('Example error:', error.message);
  }
}

// Uncomment to run the example
// example();

// Export the connector class
module.exports = AIConnector;