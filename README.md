# CloudFormation- EC2 - Static web

1. Nội dung
    - 1 Instance Ubuntu chạy service apache và deploy 1 static web
    - 1 Security group mở port 80, 22
    - Sử dụng cfn-init để xử lý các metadata và truyền tín hiệu (thành công hay thất bại) khi setup về cloudFormation.
    - Ouput xuất ra DNS public và IP public
2. Thực hiện
    - `template`
        
        ```yaml
        Parameters:
          nameOfService:
            Description: "The name of the instance"
            Type: String
          
          InstanceType:
            Description: "The instance type"
            Default: t2.micro
            AllowedValues:
              - t2.micro
              - t2.small
              - t2.medium
            Type: String
          
          KeyPair:
            Description: "The key pair"
            Type: AWS::EC2::KeyPair::KeyName
        
        Resources: 
          CoffeeShopInstance:
            Type: AWS::EC2::Instance
            Metadata:
              AWS::CloudFormation::Init:
                config:
                  packages:
                    apt:
                      apache2: []
                      unzip: []
                  services:
                    sysvinit:
                      apache2:
                        enabled: true
                        ensureRunning: true
                  commands:
                    test:
                      command: "wget https://www.tooplate.com/zip-templates/2126_antique_cafe.zip && unzip 2126_antique_cafe.zip && cp -r 2126_antique_cafe/* /var/www/html/"
                    
            Properties: 
              ImageId: ami-04ff9e9b51c1f62ca
              InstanceType: !Ref InstanceType
              KeyName: !Ref KeyPair
              Tags:
                - Key: Name
                  Value: !Ref nameOfService
              SecurityGroups:
                - !Ref MySecurityGroup
              UserData:
                "Fn::Base64": 
                  !Sub |
                    #!bin/bash
                    sudo apt update -y
                    mkdir -p /opt/aws/bin
                    wget https://s3.amazonaws.com/cloudformation-examples/aws-cfn-bootstrap-py3-latest.tar.gz
                    python3 -m easy_install --script-dir /opt/aws/bin aws-cfn-bootstrap-py3-latest.tar.gz
                    /opt/aws/bin/cfn-init -v --stack ${AWS::StackName} --resource CoffeeShopInstance --region ${AWS::Region}
                    /opt/aws/bin/cfn-signal -e $? --stack ${AWS::StackName} --resource CoffeeShopInstance --region ${AWS::Region}
            
            CreationPolicy:
              ResourceSignal:
                Count: 1
                Timeout: PT5M
              
          MySecurityGroup:
            Type: AWS::EC2::SecurityGroup
            Properties:
              GroupName: !Join [" ", ["My security group from", !Ref "AWS::StackName"]]
              GroupDescription: !Join [" ", ["My security group from", !Ref "AWS::StackName"]]
              SecurityGroupIngress:
                - IpProtocol: tcp
                  FromPort: 80
                  ToPort: 80
                  CidrIp: 0.0.0.0/0
                - IpProtocol: tcp
                  FromPort: 22
                  ToPort: 22
                  CidrIp: 0.0.0.0/0
        
        Outputs:
          PublicIp: 
            Value: !GetAtt [CoffeeShopInstance, PublicIp]
          PublicDnsName:
            Value: !GetAtt [CoffeeShopInstance, PublicDnsName]
        ```
        
        - Các parameter bao gồm:
            - `nameOfService`: Tên của EC2 instance
            - `InstanceType`: Loại instance, chỉ cho phép chọn 3 giá trị
            - `KeyPair`: Chọn key pair cho instance
        - Sử dụng `apt` để cài package apache2 và unzip
        - Enable service apache bằng `sysvinit`
        - Command để tải source code, unzip và copy source code vào thư mục `/var/www/html/`
        - `UserData` để cài đặt `cfn-bootstrap`
            - `/opt/aws/bin/cfn-init -v --stack ${AWS::StackName} --resource CoffeeShopInstance --region ${AWS::Region}`: Thực hiện xử lý các metadata (cài package, service, chạy các command,…).
            - `/opt/aws/bin/cfn-signal -e $? --stack ${AWS::StackName} --resource CoffeeShopInstance --region ${AWS::Region}`: Gửi tín hiệu thành công hay thất bại về Cloud Formation.
        - `CreationPolicy`: Để cloud formation đợi tín hiệu được gửi từ EC2 instance về trong 5 phút. (Nếu thiếu thì cloud formation sẽ không đợi mà thông báo complete, và cfn-signal sẽ không thể gửi tín hiệu)
        - `MySecurityGroup`: Tạo security group mở port 22 và 80
        - `Ouput`: Xuất ra DNS và IP public.
    - Tạo stack
        
        ![Untitled](https://i.imgur.com/blqFzf1.png)
        
        ![Untitled](https://i.imgur.com/VA3w9hO.png)
        
    - `next -> next -> create`
    - Kết quả
        
        ![Untitled](https://i.imgur.com/7tYL2D5.png)
        
        ![Untitled](https://i.imgur.com/DNGhpsp.png)
        
        ![Untitled](https://i.imgur.com/jK6oXpA.png)
        
3. Tài liệu tham khảo
    - [https://aws.amazon.com/vi/premiumsupport/knowledge-center/install-cloudformation-scripts/](https://aws.amazon.com/vi/premiumsupport/knowledge-center/install-cloudformation-scripts/)