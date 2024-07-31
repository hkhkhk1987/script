pipeline {
    agent any
    stages {
        stage('Validate YAML') {
            steps {
                script {
                    // 读取 YAML 文件内容
                    def yamlFile = 'path/to/your/file.yaml' // 修改为你的文件路径
                    def yamlContent = readFile(yamlFile)
                    
                    // 调用验证函数
                    validateYaml(yamlContent)
                }
            }
        }
    }
}

def validateYaml(yamlContent) {
    // 将 YAML 内容解析为 Map
    def yamlMap = parseYaml(yamlContent)
    
    // 用于存储 key 的列表
    def keyList = []

    if (yamlMap.containsKey('Secrets')) {
        yamlMap.Secrets.each { key ->
            // 验证 key 是否重复
            if (keyList.contains(key)) {
                error("Duplicate key found: ${key}")
            } else {
                keyList.add(key)
            }

            // 验证 key 是否符合 teamname-servicename 的规范
            if (!key.matches(/^[a-zA-Z0-9]+-[a-zA-Z0-9]+$/)) {
                error("Invalid key format: ${key}. Expected format: teamname-servicename")
            }
        }

        echo "YAML validation passed."
    } else {
        error("No 'Secrets' section found in YAML file.")
    }
}

def parseYaml(yamlContent) {
    def yamlMap = [:]
    def lines = yamlContent.readLines()
    def currentKey = null

    lines.each { line ->
        line = line.trim()
        if (line.startsWith('Secrets:')) {
            currentKey = 'Secrets'
            yamlMap[currentKey] = []
        } else if (line.startsWith('-')) {
            if (currentKey != null) {
                yamlMap[currentKey] << line.substring(1).trim().replaceAll("'", "")
            }
        }
    }

    return yamlMap
}



pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                // 检出代码
                checkout scm
            }
        }
        
        stage('Validate YAML') {
            steps {
                script {
                    def yamlFile = 'config.yaml'
                    
                    // 读取 YAML 文件内容
                    def yamlContent = readFile(yamlFile)
                    
                    // 解析 YAML 文件
                    def yaml = new groovy.yaml.YamlSlurper().parseText(yamlContent)

                    // 检查键是否重复
                    def keys = []
                    def duplicates = []

                    def collectKeys
                    collectKeys = { map, parentKey = '' ->
                        map.each { k, v ->
                            def fullKey = parentKey ? "${parentKey}.${k}" : k
                            if (keys.contains(fullKey)) {
                                duplicates << fullKey
                            } else {
                                keys << fullKey
                            }
                            if (v instanceof Map) {
                                collectKeys(v, fullKey)
                            }
                        }
                    }

                    collectKeys(yaml)

                    if (duplicates) {
                        error "Duplicate keys found: ${duplicates.join(', ')}"
                    } else {
                        echo "YAML validation passed."
                    }
                }
            }
        }
    }
}


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
