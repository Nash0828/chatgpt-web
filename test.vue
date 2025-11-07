class MockDataGenerator {
    static generateMockData() {
        const nodes = [];
        const links = [];
        
        // 生成50个节点
        for (let i = 0; i < 50; i++) {
            nodes.push(this.generateNode(i));
        }
        
        // 生成约200条边
        const linkCount = 200;
        for (let i = 0; i < linkCount; i++) {
            const link = this.generateLink(nodes, links);
            if (link) {
                links.push(link);
            }
        }
        
        return { nodes, links };
    }
    
    static generateNode(index) {
        const types = ['server', 'client', 'router', 'switch', 'database', 'firewall', 'loadbalancer'];
        const statuses = ['online', 'offline', 'degraded'];
        const regions = ['us-east', 'us-west', 'eu-central', 'ap-south', 'cn-north'];
        
        return {
            id: this.generateUUID(),
            name: `${this.getNodeTypeName(types[index % types.length])} ${Math.floor(index / 10) + 1}-${(index % 10) + 1}`,
            type: types[index % types.length],
            status: statuses[index % statuses.length],
            region: regions[index % regions.length],
            cpu: Math.floor(Math.random() * 100),
            memory: Math.floor(Math.random() * 100),
            disk: Math.floor(Math.random() * 100),
            tags: this.generateTags(index)
        };
    }
    
    static generateLink(nodes, existingLinks) {
        const maxAttempts = 50; // 防止无限循环
        let attempts = 0;
        
        while (attempts < maxAttempts) {
            const sourceIndex = Math.floor(Math.random() * nodes.length);
            let targetIndex = Math.floor(Math.random() * nodes.length);
            
            // 确保不连接到自身
            if (sourceIndex === targetIndex) {
                attempts++;
                continue;
            }
            
            const sourceId = nodes[sourceIndex].id;
            const targetId = nodes[targetIndex].id;
            
            // 检查边是否已存在
            const linkExists = existingLinks.some(link => 
                (link.source === sourceId && link.target === targetId) ||
                (link.source === targetId && link.target === sourceId)
            );
            
            if (!linkExists) {
                return {
                    source: sourceId,
                    target: targetId,
                    bandwidth: this.generateBandwidth(),
                    latency: Math.floor(Math.random() * 100) + 1,
                    status: Math.random() > 0.1 ? 'active' : 'degraded'
                };
            }
            
            attempts++;
        }
        
        return null; // 如果找不到可用的连接
    }
    
    static generateUUID() {
        return 'xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx'.replace(/[xy]/g, function(c) {
            const r = Math.random() * 16 | 0;
            const v = c == 'x' ? r : (r & 0x3 | 0x8);
            return v.toString(16);
        });
    }
    
    static getNodeTypeName(type) {
        const names = {
            server: 'Server',
            client: 'Client',
            router: 'Router',
            switch: 'Switch',
            database: 'DB',
            firewall: 'Firewall',
            loadbalancer: 'LB'
        };
        return names[type] || 'Node';
    }
    
    static generateBandwidth() {
        const bandwidths = ['10Mbps', '100Mbps', '1Gbps', '10Gbps', '40Gbps'];
        return bandwidths[Math.floor(Math.random() * bandwidths.length)];
    }
    
    static generateTags(index) {
        const allTags = ['production', 'development', 'staging', 'backend', 'frontend', 'database', 'network', 'security', 'monitoring', 'logging'];
        const tagCount = Math.floor(Math.random() * 3) + 1; // 1-3个标签
        const tags = [];
        
        for (let i = 0; i < tagCount; i++) {
            const tag = allTags[Math.floor(Math.random() * allTags.length)];
            if (!tags.includes(tag)) {
                tags.push(tag);
            }
        }
        
        return tags;
    }
}