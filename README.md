class Connection():
#每个连接对象都要记录该连接的权重。
#主要职责是记录连接的权重，以及这个连接所关联的上下游节点。
    def __init__(self, conn_up_node, conn_down_node):
        '''
        初始化一条连接（即章鱼的触脚)
        conn_up_node: 连接的上游节点(某只章鱼头部)，如<Node object at 0x111c>，即Node类的一个实例。
        conn_down_node: 连接的下游节点(这只章鱼的某条触脚触及的节点)，如<Node object at 0x111c>
        '''
        self.conn_up_node = conn_up_node
        self.conn_down_node = conn_down_node
        self.weight = random.uniform(-0.1, 0.1)
        self.gradient = 0.0
    def calc_gradient(self):
        #计算梯度，《机器学习》P103页【3】式
        self.gradient = self.conn_down_node.delta * self.conn_up_node.output
    def get_gradient(self):
        #获取当前连接的梯度
        return self.gradient
    def update_weight(self, rate):
        #根据梯度下降法更新此连接的权重
        self.calc_gradient()
        self.weight += rate * self.gradient



class Network():
#提供API接口。它由若干层对象组成以及连接对象组成
    def __init__(self, layers):
        #初始化一个全连接神经网络
        #layers: 描述神经网络每层节点数，如[7, 3, 10]
        self.connections = []
        self.layers = []
        layer_count = len(layers)
        node_count = 0
        for i in range(layer_count):
            self.layers.append(Layer(i, layers[i]))
        #print(self.layers)
        #[<Layer object at 0x10f8>, <Layer object at 0x11a8>, <Layer object at 0x11p0>]
        #self.layers[0]是输入层，self.layers[1]是隐藏层，self.layers[2]是输出层。
        #print('输出层节点：',self.layers[2].nodes)
        # [<Node object at 0x111c>, <Node object at 0x111c>, <Node object at 0x111u>, … 共11个]
        for layer in range(layer_count - 1):
            connections = [Connection(conn_up_node, conn_down_node) 
                for conn_up_node in self.layers[layer].nodes
                for conn_down_node in self.layers[layer + 1].nodes[:-1]
            ]
            '''
            以上两个for不是平级，而是上下级嵌套关系。
            想象一下，神经网络每一层都是有很多章鱼组成，每一层章鱼的头部摆在左，触脚摆在右。
            上游节点指的是某只章鱼的头部，下游节点指的是这只章鱼的各条触脚触及的节点；
            每一条触脚就是一个Connection类（只有层与层之间存在触脚）。
            为什么下游节点要剔除掉最后一个节点？因为每一层的最后一个节点默认指定是
            偏置项节点(输出恒为1的参数节点)，这种节点就不要分配触脚与之连接了，浪费资源。
            '''
            #print('当layer为1时，即隐藏层到输出层所有连接：',connections)
            #[<Connection object at 0x10b9>, <Connection object at 0x10b9>,…共40个]
            for conn in connections:
                self.connections.append(conn)

            #对于某条连接conn，既要赠付给他所连的下游节点，同时也要赠付给他所连的上游节点。
                conn.conn_down_node.append_upstream_connection(conn)
                conn.conn_up_node.append_downstream_connection(conn)
    def train(self, labels, data_set, rate, iteration):
        #训练神经网络
        #labels: 数组，训练样本标签。每个元素是一个样本的标签。
        #data_set: 二维数组，训练样本特征。每个元素是一个样本。
        for i in range(iteration):
            for d in range(len(data_set)): #len(data_set)是样本个数：6个
                #针对某个样本
                self.predict(data_set[d])
                self.calc_delta(labels[d])
                self.update_weight(rate)
    def calc_delta(self, label):
        #计算每个节点的delta
        output_nodes = self.layers[-1].nodes #输出层的所有节点
        for i in range(len(label)):
            output_nodes[i].calc_shuchuceng_jiedian_delta(label[i])
        for layer in self.layers[-2::-1]: #反向计算
            #先对隐藏层，后对输入层，进行以下计算
            for node in layer.nodes:
                node.calc_feishuchuceng_jiedian_delta()
    def update_weight(self, rate):
        #更新所有下游连接权重，输出层没有下游连接，所以不考虑输出层
        for layer in self.layers[:-1]:
            for node in layer.nodes:
                for conn in node.downstream:
                    conn.update_weight(rate)
    
    def predict(self, sample):
        #根据输入的样本预测输出值
        #sample: 数组，某个样本的特征向量，也就是网络的输入向量
        self.layers[0].set_shuruceng_output(sample) #输入层里的各个节点输出
        for i in range(1, len(self.layers)):
            self.layers[i].calc_output() #非输入层里的各个节点输出
        
        #返回输出层(self.layers[-1])里各个节点的输出值组成的列表/向量。
        return list(map(lambda node: node.output, self.layers[-1].nodes[:-1]))
    
    #以下两个方法是附加的，用于梯度比较，没什么意义可删除，非组成神经网络的必要方法
    def calc_gradient(self):
        #计算每个连接的梯度
        for layer in self.layers[:-1]:
            for node in layer.nodes:
                for conn in node.downstream:
                    conn.calc_gradient()
    def get_gradient(self, sample,label):
        #获得网络在一个样本下，每个连接上的梯度
        self.predict(sample)
        self.calc_delta(label)
        self.calc_gradient()
