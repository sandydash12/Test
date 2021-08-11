
public class ForwardMail {
    
	static Gmail service = null;
	private static final String USER_ID = "me";
	private static File filePath = new File(System.getProperty("user.dir") + File.separator + "src" + File.separator
			+ "test" + File.separator + "resources" +
			// File.separator + "credentials" +
			File.separator + "Credentials.json");
	private static final JsonFactory JSON_FACTORY = JacksonFactory.getDefaultInstance();
	private static final String APPLICATION_NAME = "*************";


	
	public static void main(String[] args) {
		
		try {
			replyMail("subject:Sandbox");
		} catch (IOException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		} catch (GeneralSecurityException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
	}

    public static void replyMail(String query) throws IOException, GeneralSecurityException {
       

    	Gmail service = getService();
		List<Message> messages = listMessagesMatchingQuery(service, USER_ID, query);
		Message message = getMessage(service, USER_ID, messages, 0);
        List<String> headers = Arrays.asList("Subject", "To", "From", "Message-ID", "References");

        Map<String, String> replyHeaders = message.getPayload().getHeaders()
                .stream()
                .filter((header) -> headers.contains(header.getName())
                ).collect(Collectors.toMap(MessagePartHeader::getName, MessagePartHeader::getValue));


        try {
            Message replayEmail = createReplayEmail(replyHeaders,message.getThreadId());
            service.users()
                    .messages()
                    .send(USER_ID,replayEmail)
                    .execute();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
    

    
    public static List<Message> listMessagesMatchingQuery(Gmail service, String userId, String query)
			throws IOException {
		ListMessagesResponse response = service.users().messages().list(userId).setQ(query).execute();
		List<Message> messages = new ArrayList<Message>();
		while (response.getMessages() != null) {
			messages.addAll(response.getMessages());
			if (response.getNextPageToken() != null) {
				String pageToken = response.getNextPageToken();
				response = service.users().messages().list(userId).setQ(query).setPageToken(pageToken).execute();
			} else {
				break;
			}
		}
		return messages;
	}
    
	public static Message getMessage(Gmail service, String userId, List<Message> messages, int index)
			throws IOException {
		Message message = service.users().messages().get(userId, messages.get(index).getId()).execute();
		return message;
	}
    
    
    
    
    public static Message createReplayEmail(Map<String, String> headers, String threadId)
            throws MessagingException, IOException {
        MimeMessage email = createMimeMessage(headers);

        ByteArrayOutputStream buffer = new ByteArrayOutputStream();
        email.writeTo(buffer);
        byte[] bytes = buffer.toByteArray();
        String encodedEmail = Base64.getUrlEncoder().encodeToString(bytes);
        Message message = new Message();
        message.setThreadId(threadId);
        message.setRaw(encodedEmail);
        return message;
    }

    private static MimeMessage createMimeMessage(Map<String, String> headers) throws MessagingException {
        Properties props = new Properties();
        Session session = Session.getDefaultInstance(props, null);
        MimeMessage email = new MimeMessage(session);

        email.setFrom(new InternetAddress(headers.get("To")));
        email.addRecipient(javax.mail.Message.RecipientType.TO,
                new InternetAddress(headers.get("From")));
        email.setSubject("Re: " + headers.get("Subject"));

        MimeBodyPart mimeBodyPart = new MimeBodyPart();
        mimeBodyPart.setContent("Re Re Re...", "text/plain");

        email.addHeader("In-Reply-To",headers.get("Message-ID"));
        email.addHeader("References",headers.get("References"));

        Multipart multipart = new MimeMultipart();
        multipart.addBodyPart(mimeBodyPart);
        email.setContent(multipart);
        return email;
    }
}
