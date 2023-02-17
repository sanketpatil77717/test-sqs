# test-sqs

Let's assume that you have two types of messages: TypeA and TypeB. Both message types have different structures and content. Here's an example of the JSON format for each message type:

TypeA message:
{
  "type": "TypeA",
  "id": 1234,
  "name": "John Doe",
  "email": "johndoe@example.com"
}


TypeB message:
{
  "type": "TypeB",
  "order_id": "5678",
  "customer_id": "ABCD",
  "total_amount": 100.0
}


Now, you can define a strategy interface or abstract class for publishing messages to SQS:
public interface MessagePublisher {
    void publish(String message);
}



Next, you can create two concrete implementations for each message type, TypeAPublisher and TypeBPublisher, respectively:


@Component
public class TypeAPublisher implements MessagePublisher {
    @Override
    public void publish(String message) {
        // Parse the message JSON and extract the relevant data for TypeA message
        int id = ...;
        String name = ...;
        String email = ...;

        // Use the AWS SDK to publish the message to SQS
        AmazonSQS sqs = AmazonSQSClientBuilder.defaultClient();
        SendMessageRequest request = new SendMessageRequest()
                .withQueueUrl("TypeA-Queue-URL")
                .withMessageBody(message);
        sqs.sendMessage(request);
    }
}

@Component
public class TypeBPublisher implements MessagePublisher {
    @Override
    public void publish(String message) {
        // Parse the message JSON and extract the relevant data for TypeB message
        String orderId = ...;
        String customerId = ...;
        double totalAmount = ...;

        // Use the AWS SDK to publish the message to SQS
        AmazonSQS sqs = AmazonSQSClientBuilder.defaultClient();
        SendMessageRequest request = new SendMessageRequest()
                .withQueueUrl("TypeB-Queue-URL")
                .withMessageBody(message);
        sqs.sendMessage(request);
    }
}



Finally, you can create a MessageHandler class that receives the message type and message content and uses the appropriate MessagePublisher implementation to publish the message to SQS:


@Component
public class MessageHandler {
    @Autowired
    private Map<String, MessagePublisher> publishers;

    public void handleMessage(String messageType, String message) {
        MessagePublisher publisher = publishers.get(messageType);
        if (publisher == null) {
            // handle the unknown message type
        } else {
            publisher.publish(message);
        }
    }
}



To use this MessageHandler in your Spring Boot application, you can define a REST endpoint that receives the message type and content as a JSON request:
@RestController
public class MessageController {
    @Autowired
    private MessageHandler messageHandler;

    @PostMapping("/messages")
    public void handleMessage(@RequestBody String requestBody) {
        try {
            ObjectMapper mapper = new ObjectMapper();
            JsonNode root = mapper.readTree(requestBody);
            String messageType = root.path("type").asText();
            String message = mapper.writeValueAsString(root);

            messageHandler.handleMessage(messageType, message);
        } catch (IOException e) {
            // handle the error
        }
    }
}




If the messages don't have a type attribute that you can use to identify their type, you'll need to come up with a different strategy to determine their type based on other attributes.

One approach is to define a set of rules or conditions that determine the message type based on the presence or absence of specific fields. For example, you might look for the presence of a customerId field to determine that a message is of TypeA, and look for the presence of an orderId field to determine that a message is of TypeB.

Here's an example implementation of the handleMessage method in the MessageController that uses this approach:

@PostMapping("/messages")
public void handleMessage(@RequestBody String requestBody) {
    try {
        ObjectMapper mapper = new ObjectMapper();
        JsonNode root = mapper.readTree(requestBody);
        String messageType = getMessageType(root);

        String message = mapper.writeValueAsString(root);
        messageHandler.handleMessage(messageType, message);
    } catch (IOException e) {
        // handle the error
    }
}

private String getMessageType(JsonNode root) {
    if (root.has("customerId")) {
        return "TypeA";
    } else if (root.has("orderId")) {
        return "TypeB";
    } else {
        return "Unknown";
    }
}


