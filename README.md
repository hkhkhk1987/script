# script
pipeline {
    agent any

    stages {
        stage('Prepare') {
            steps {
                // Checkout your repository containing the CSV file
                git 'https://your-repo-url.git'
            }
        }
        
        stage('Process CSV') {
            steps {
                script {
                    // Define the input and output file names
                    def inputFile = 'your_file.csv'
                    def outputFile = 'your_file_processed.csv'
                    
                    // Read the input CSV file
                    def input = new File(inputFile)
                    def lines = input.readLines()
                    
                    // Split the first line (header) to get the column names
                    def header = lines[0].split(',')
                    def eimIndex = header.indexOf('eim')
                    
                    // Create a new list to hold the processed lines
                    def processedLines = []
                    
                    // Add the new header with 'eim2'
                    processedLines << (header + 'eim2').join(',')
                    
                    // Process each line
                    lines.tail().each { line ->
                        def values = line.split(',')
                        def eimValue = values[eimIndex].toDouble()
                        def eim2Value = eimValue * 2
                        processedLines << (values + eim2Value).join(',')
                    }
                    
                    // Write the processed lines to the output CSV file
                    new File(outputFile).withWriter { writer ->
                        processedLines.each { writer.println(it) }
                    }
                }
            }
        }
        
        stage('Archive Results') {
            steps {
                // Archive the processed CSV file
                archiveArtifacts artifacts: 'your_file_processed.csv', allowEmptyArchive: true
            }
        }
    }
}



pipeline {
    agent any

    environment {
        CSV_FILE = 'path/to/your/file.csv'
        HTML_FILE = 'path/to/your/file.html'
        RECIPIENT_EMAIL = 'recipient@example.com'
    }

    stages {
        stage('Convert CSV to HTML') {
            steps {
                script {
                    // Read the CSV file
                    def csvContent = readFile(file: CSV_FILE)

                    // Convert CSV to HTML
                    def csvLines = csvContent.split('\n')
                    def headers = csvLines[0].split(',')
                    def rows = csvLines[1..-1]

                    def htmlContent = """
                        <html>
                        <head>
                            <style>
                                table {
                                    width: 100%;
                                    border-collapse: collapse;
                                }
                                th {
                                    background-color: blue;
                                    color: white;
                                    border: 1px solid black;
                                    padding: 8px;
                                    text-align: left;
                                }
                                td {
                                    border: 1px solid black;
                                    padding: 8px;
                                    text-align: left;
                                }
                            </style>
                        </head>
                        <body>
                            <table>
                                <thead>
                                    <tr>
                    """
                    headers.each { header ->
                        htmlContent += "<th>${header}</th>"
                    }
                    htmlContent += """
                                    </tr>
                                </thead>
                                <tbody>
                    """
                    rows.each { row ->
                        htmlContent += "<tr>"
                        row.split(',').each { cell ->
                            htmlContent += "<td>${cell}</td>"
                        }
                        htmlContent += "</tr>"
                    }
                    htmlContent += """
                                </tbody>
                            </table>
                        </body>
                        </html>
                    """

                    // Write the HTML content to a file
                    writeFile file: HTML_FILE, text: htmlContent
                }
            }
        }

        stage('Send Email') {
            steps {
                script {
                    emailext(
                        subject: 'CSV to HTML Report',
                        body: 'Please find the attached HTML report converted from CSV.',
                        to: RECIPIENT_EMAIL,
                        attachFiles: HTML_FILE,
                        mimeType: 'text/html'
                    )
                }
            }
        }
    }
}
