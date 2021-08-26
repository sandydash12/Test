    
	static Gmail service = null;
	private static final String USER_ID = "me";
	private static File filePath = new File(System.getProperty("user.dir") + File.separator + "src" + File.separator
			+ "test" + File.separator + "resources" +
			// File.separator + "credentials" +
			File.separator + "Credentials.json");
	private static final JsonFactory JSON_FACTORY = JacksonFactory.getDefaultInstance();
	private static final String APPLICATION_NAME = "*********";


	
	public static void main(String[] args) {
		
		try {
			replyMail("***********");
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
    
    
    @SuppressWarnings("deprecation")
	public static Gmail getService() throws IOException, GeneralSecurityException {

		InputStream in = new FileInputStream(filePath); // Read credentials.json
		GoogleClientSecrets clientSecrets = GoogleClientSecrets.load(JSON_FACTORY, new InputStreamReader(in));

		Credential authorize = new GoogleCredential.Builder().setTransport(GoogleNetHttpTransport.newTrustedTransport())
				.setJsonFactory(JSON_FACTORY)
				.setClientSecrets(clientSecrets.getDetails().getClientId().toString(),
						clientSecrets.getDetails().getClientSecret().toString())
				.build().setAccessToken(getAccessToken()).setRefreshToken(GmailAPICredentials.refresh_token);

		// Build a new authorized API client service.
		final NetHttpTransport HTTP_TRANSPORT = GoogleNetHttpTransport.newTrustedTransport();
		Gmail service = new Gmail.Builder(HTTP_TRANSPORT, JSON_FACTORY, authorize).setApplicationName(APPLICATION_NAME)
				.build();
		return service;
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
    
    
    public static HashMap<String, String> getGmailData(String query) {
		try {
			
			Gmail service = getService();
			List<Message> messages = listMessagesMatchingQuery(service, USER_ID, query);
			Message message = getMessage(service, USER_ID, messages, 0);
			//System.out.println(message.toPrettyString());
			JsonPath jp = new JsonPath(message.toString());
			String subject = jp.getString("payload.headers.find { it.name == 'Subject' }.value");
			String messageId = jp.getString("payload.headers.find { it.name == 'Message-ID' }.value");
			String body = new String(Base64.getDecoder().decode(jp.getString("payload.parts[0].body.data")));
			String from = jp.getString("payload.headers.find { it.name == 'From' }.value");
			String link = null;
			String ThreadId = jp.getString("threadId");
			String arr[] = body.split("\n");
			for (String s : arr) {
				s = s.trim();
				if (s.startsWith("http") || s.startsWith("https")) {
					link = s.trim();
				}
			}
			HashMap<String, String> hm = new HashMap<String, String>();
			hm.put("subject", subject);
			hm.put("body", body);
			hm.put("link", link);
			hm.put("MessageId", messageId);
			hm.put("threadId", ThreadId);
			hm.put("ReplyTo", from);
			return hm;

		} catch (Exception e) {
			System.out.println("email not found....");
			throw new RuntimeException(e);
		}
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

        MimeBodyPart mimeBodyPart = new MimeBodyPart();
        mimeBodyPart.setContent("Re Re Re...", "text/plain");

        email.addHeader("In-Reply-To",headers.get("Message-ID"));
        email.addHeader("References",headers.get("References"));
        email.addHeader("Subject", headers.get("Subject"));

        Multipart multipart = new MimeMultipart();
        multipart.addBodyPart(mimeBodyPart);
        email.setContent(multipart);
        return email;
    }
}

