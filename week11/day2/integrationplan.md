Context:
As a Senior Architect visual regression agent application should be integrated with Moderation layer refering the Postman collection AI-shield.postman_Collection.json

Instructions:
[MANDATORY] Incorporate API changes first and then Proceed with UI
[MANDATORY] Analyze the Postman collection AI-shield.postman_Collection.json to identify the endpoints, request methods, and expected responses that need to be integrated with the Moderation layer.
[MANDATORY] If the action is block from the moderation layer, call should not be intitiated to LLM for processing.
[MANDATORY] Toggle on/off should be provided for the integration with the Moderation layer in the .env file.
[MANDATORY] For every request run the /api/moderate endpoint to check if the content is allowed or blocked before proceeding with the request to the LLM.If allowed
Proceed with Generation
[MANDATORY] If blocked, return a user-friendly error message indicating that the content is not allowed and provide guidance on what to do next.
[MANDATORY] Display reason with the detector details