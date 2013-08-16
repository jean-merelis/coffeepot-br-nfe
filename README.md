coffeepot-br-nfe
================


Project to be used in Brazil for to sign and to send fiscal documents of the project NF-e of the federal government.

Projeto para comunicação com os portais do sefaz, para envio de NF-e, envio de Eventos de NF-e, envio de Inutilização e consultas.

Javadoc disponível em: 
  http://jean-merelis.github.io/coffeepot-br-nfe/
  
O jar do projeto pode ser baixado no sourceforce: 
    https://sourceforge.net/projects/coffeepotbrnfe/  
  
 
Exemplo:

A classe NFeFacade é uma fachada para os serviços relacionados a NF-e desta API.
Neste exemplo será mostrado parte do código da classe de testes da NFeFacade.
É necessário ter um certificado válido para realização dos testes.
Importante: Comunicações com a SEFAZ irâo ocorrer!!


    NFeFacade instance;

    @BeforeClass
    public static void config() throws Exception {
        //Primeira coisa a se fazer é configurar o ambiente e criar a instancia da fachada.
    
        boolean linux = System.getProperty("os.name").equalsIgnoreCase("linux");
        String pwd = "123";
        String filePath;

        if (linux) {
            filePath = System.getProperty("user.home");
        } else {
            filePath = "C:\\";
        }

        // carrega o certificado
        KeyStore ks = KeyStore.getInstance("PKCS12");
        ks.load(new FileInputStream(filePath + File.separator + "certificado_testes.pfx"), pwd.toCharArray());
        String alias = ks.aliases().nextElement();

        //configura o ambiente
        Environment environment = new Environment(Environment.Type.HOMOLOGATION);
        CertificateData certData = new CertificateData(ks, alias, pwd.toCharArray());
        certData.setCacerts(new File(filePath + File.separator + "NFeCacerts"));
        environment.setCertificateData(certData);
        environment.setUfOfEmitter(UF.ES);
        environment.setDataVersion("2.00");

        instance = new NFeFacade(environment);
    }
    
    /**
    *Consultar status do serviço.
    */
    @Test
    public void testQueryServiceStatus_0args() throws Exception {
        System.out.println();
        System.out.println("=== testQueryServiceStatus_0args ===");

        StatusQueryResult result = instance.queryServiceStatus();

        assertNotNull(result);
        
        System.out.println("código do status= " + result.getStatusCode());
        System.out.println("motivo= " + result.getMotive());
        System.out.println("código da UF= " + result.getStateCode());
        System.out.println("data/hora recebimento= " + result.getDateTimeOfReceipt());
        System.out.println("resposta original: \n" + result.getOriginalResponse());
    }
    
    /**
     * Envio de nfe.
     */
    @Test
    public void testSendBatchOfNFe() throws Exception {
        System.out.println();
        System.out.println("=== testSendBatchOfNFe ===");
        
        String xml = "Coloque o XML da sua NFe aqui";
        
        //assina o XML
        XmlSignResult xmlSigned = instance.signNFe(xml);
        
        //valida o XML, como o schema PL_006r
        ValidationResultsHandler validateResult = instance.validateXmlNFe(xmlSigned.getXml(), SchemaVersion.getTheLatestSchema());

        //processa o retorno da validação
        if (validateResult != null) {
            if (validateResult.hasErrors()) {
                System.out.println("XML inválido:");
                for (String s : validateResult.getErrors()) {
                    System.out.println("Erro: " + s);
                }
                for (String s : validateResult.getFatalErros()) {
                    System.out.println("Erro fatal: " + s);
                }
            }
            if (validateResult.hasWarnings()) {
                for (String s : validateResult.getWarnings()) {
                    System.out.println("Avisos: " + s);
                }
            }
        }

        List<String> loteNFe = new ArrayList<>();
        
        //remove a tag de versão do XML
        String xmlNFe =xmlSigned.getXml().replace("<?xml version=\"1.0\" encoding=\"UTF-8\"?>", "");
        
        //adiona o xml da NFe no lote
        loteNFe.add(xmlNFe);

        //envia o lote de NF-e para SEFAZ
        BatchSubmissionResult result = instance.sendBatchOfNFe(3, loteNFe);

        assertNotNull(result);

        //Processa o resultado recebido
        if (result.successfullyReceived()) {
            System.out.println("Lote recebido com sucesso!");
            System.out.println("Número do recibo: " + result.getReceiptNumber());
            System.out.println("versão aplic: " + result.getApplicationVersion());
            System.out.println("Data/hora recebimento: " + result.getDateTimeOfReceipt());
            System.out.println("Ambiente: " + result.getEnvironmentType());
            System.out.println("Tempo médio: " + result.getAverageTime());

        } else {
            System.out.println("Lote não recebido com sucesso :(");
            System.out.println("Código: " + result.getStatusCode());
            System.out.println("Motivo: " + result.getMotive());
        }
        assertTrue(true);
    }
    
    /**
     * Consulta um lote de NF-e enviado
     */
    @Test
    public void testQueryBatchOfNFe() throws Exception {
        System.out.println();
        System.out.println("=== testQueryBatchOfNFe ===");

        //Informe um número de recibo de lote válido
        Long receiptNumber = 324000700123255L;

        //efetua a consulta na SEFAZ
        BatchQueryResult result = instance.queryBatchOfNFe(receiptNumber);

        assertNotNull(result);

        System.out.println("código do status=" + result.getStatusCode());
        System.out.println("motivo=" + result.getMotive());
        System.out.println("código da UF=" + result.getStateCode());

        System.out.println("Número do recibo:" + result.getReceiptNumber());

        //Exibe o resultado das notas processadas na SEFAZ
        if (result.getProtocolProcessingList() != null) {
            for (ProtocolProcessing p : result.getProtocolProcessingList()) {
                System.out.println("--------------------------------");
                System.out.println(p.getAccessKey());
                System.out.println(p.getDigestValue());
                System.out.println(p.getProtocolNumber());
                System.out.println(p.getMotive());
                System.out.println(p.getStatusCode());
                System.out.println(p.getDateTimeOfReceipt());
            }
            System.out.println("--------------------------------");
        }

        assertTrue(result.getReceiptNumber().equals(receiptNumber));
    }
    
    
    
Eventos de cancelamento,CCe e Manifestação serão mostrados em outra oportunidade.


  
