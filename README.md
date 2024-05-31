
# addImageteam03
function addImage(tokenValue,id) {
    const bearerToken = tokenValue;
    function extractNormalTextFromGoogleDoc() {
      const docId = id ; // Replace with your Google Doc ID
      const doc = DocumentApp.openById(docId);
      const body = doc.getBody();
      const paragraphs = body.getParagraphs();
      const result = [];
      paragraphs.forEach(paragraph => {
        const text = paragraph.getText();
        const paragraphStyle = paragraph.getHeading();
        if (text.trim() !== '' && paragraphStyle == DocumentApp.ParagraphHeading.NORMAL) {
          result.push(paragraph);
        }
      });
      Logger.log(result);
      return result;
    }
    function generateImage(prompt) {
      const apiUrl = 'https://api.monsterapi.ai/v1/generate/txt2img';
      const headers = {
        'Accept': 'application/json',
        'Authorization': `Bearer ${bearerToken}`
      };
      const data = {
        'prompt': prompt,
        'aspect_ratio': 'portrait',
        'guidance_scale': '12.5'
      };
      const options = {
        'method': 'post',
        'headers': headers,
        'payload': JSON.stringify(data),
        'contentType': 'application/json',
        'muteHttpExceptions': true  // Enable to examine full response
      };
      const response = UrlFetchApp.fetch(apiUrl, options);
      const responseCode = response.getResponseCode();
      if (responseCode !== 200) {
        Logger.log(`Error ${responseCode}: ${response.getContentText()}`);
        throw new Error(`Request failed with response code ${responseCode}`);
      }
      const responseJson = JSON.parse(response.getContentText());
      console.log("API Response:", responseJson);
      const processId = responseJson['process_id'];
      const statusUrl = responseJson['status_url'];
      return [processId, statusUrl];
    }
    function getImageResult(statusUrl) {
      const headers = {
        'Authorization': `Bearer ${bearerToken}`,
        'Accept': 'application/json'
      };
      while (true) {
        const options = {
          'method': 'get',
          'headers': headers,
          'muteHttpExceptions': true  // Enable to examine full response
        };
        const response = UrlFetchApp.fetch(statusUrl, options);
        const responseCode = response.getResponseCode();
        if (responseCode !== 200) {
          Logger.log(`Error ${responseCode}: ${response.getContentText()}`);
          throw new Error(`Request failed with response code ${responseCode}`);
        }
        const responseJson = JSON.parse(response.getContentText());
        console.log("Status Response:", responseJson);
        const status = responseJson['status'];
        if (status === 'COMPLETED') {
          const result = responseJson['result'];
          if (result) {
            return result['output'];
          }
        } else if (status === 'FAILED') {
          throw new Error("Image generation failed");
        }
        Utilities.sleep(5000);
      }
    }
    function insertImageAfterParagraph(paragraph, imageUrl) {
      const response = UrlFetchApp.fetch(imageUrl);
      const blob = response.getBlob();
      const parent = paragraph.getParent();
      const paragraphIndex = parent.getChildIndex(paragraph);
      // Insert a new paragraph after the current one
      const newParagraph = parent.insertParagraph(paragraphIndex + 1, "");
      const image = newParagraph.appendInlineImage(blob);
      // Resize the image to 50x50
      image.setWidth(150);
      image.setHeight(150);
    }
    function main() {
      try {
        const paragraphs = extractNormalTextFromGoogleDoc();
        paragraphs.forEach(paragraph => {
          const text = paragraph.getText();
          const prompt = text;
          Logger.log(prompt);
          const [processId, statusUrl] = generateImage(prompt);
          console.log(`Process ID: ${processId}`);
          console.log(`Status URL: ${statusUrl}`);
          const imageUrls = getImageResult(statusUrl);
          console.log(`Generated Image URLs: ${imageUrls}`);
          if (imageUrls.length > 0) {
            insertImageAfterParagraph(paragraph, imageUrls[0]); // Insert the first image URL after the paragraph
            console.log(`Image inserted after paragraph: "${text}"`);
          }
        });
      } catch (e) {
        console.error(`An error occurred: ${e.message}`);
        return e.message
      }
    }
    return main();
  }
