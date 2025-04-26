import SpriteKit
import GameplayKit

class GameScene: SKScene, SKPhysicsContactDelegate {

    //This was made by my friend in China, if the language you cannot know, please go to https://translate.google.com/?sl=auto&tl=en&op=translate.
    
    var bluePlayer: SKSpriteNode!
    var redPlayer: SKSpriteNode!

    var blueHP = 1000
    var redHP = 1000
    var blueCombo = 0
    var redCombo = 0

    var hpLabelBlue: SKLabelNode!
    var hpLabelRed: SKLabelNode!

    var keysPressed = Set<Character>()

    var isBlueKnockedBack = false
    var isRedKnockedBack = false

    // MARK: - 新增的攻击冷却属性
    var isBlueAttackOnCooldown = false
    var isRedAttackOnCooldown = false
    let attackCooldownDuration: TimeInterval = 1.0 // 冷却时间2秒

    // MARK: - didMove(to view:)
    override func didMove(to view: SKView) {
        physicsWorld.contactDelegate = self

        backgroundColor = .white
        physicsWorld.gravity = CGVector(dx: 0, dy: -6.0)

        self.physicsBody = SKPhysicsBody(edgeLoopFrom: self.frame)
        self.physicsBody?.isDynamic = false

        let ground = SKSpriteNode(color: .gray, size: CGSize(width: size.width, height: 50))
        ground.position = CGPoint(x: size.width / 2, y: 25)
        ground.physicsBody = SKPhysicsBody(rectangleOf: ground.size)
        ground.physicsBody?.isDynamic = false
        addChild(ground)

        bluePlayer = SKSpriteNode(color: .blue, size: CGSize(width: 40, height: 100))
        bluePlayer.position = CGPoint(x: size.width * 0.3, y: 100)
        bluePlayer.physicsBody = SKPhysicsBody(rectangleOf: bluePlayer.size)
        configurePhysics(for: bluePlayer)
        addChild(bluePlayer)

        redPlayer = SKSpriteNode(color: .red, size: CGSize(width: 40, height: 100))
        redPlayer.position = CGPoint(x: size.width * 0.7, y: 100)
        redPlayer.physicsBody = SKPhysicsBody(rectangleOf: redPlayer.size)
        configurePhysics(for: redPlayer)
        addChild(redPlayer)

        hpLabelBlue = SKLabelNode(text: "蓝方 HP: \(blueHP)")
        hpLabelBlue.fontSize = 18
        hpLabelBlue.fontColor = .blue
        hpLabelBlue.position = CGPoint(x: 120, y: size.height - 40)
        hpLabelBlue.horizontalAlignmentMode = .left
        addChild(hpLabelBlue)

        hpLabelRed = SKLabelNode(text: "红方 HP: \(redHP)")
        hpLabelRed.fontSize = 18
        hpLabelRed.fontColor = .red
        hpLabelRed.position = CGPoint(x: size.width - 120, y: size.height - 40)
        hpLabelRed.horizontalAlignmentMode = .right
        addChild(hpLabelRed)
    }

    func configurePhysics(for node: SKSpriteNode) {
        guard let body = node.physicsBody else { return }
        body.mass = 1.0
        body.allowsRotation = false
        body.restitution = 0.0
        body.friction = 0.2
        body.linearDamping = 0.5
        body.categoryBitMask = 0x1 << 0
        body.collisionBitMask = 0xFFFFFFFF
        body.contactTestBitMask = 0xFFFFFFFF
    }

    // MARK: - keyDown(with event:)
    override func keyDown(with event: NSEvent) {
        guard let chars = event.characters else { return }

        for char in chars {
            if !keysPressed.contains(char) {
                keysPressed.insert(char)

                switch char {
                case "w": // 蓝方跳跃
                    if isOnGround(bluePlayer) && !isBlueKnockedBack {
                        bluePlayer.physicsBody?.applyImpulse(CGVector(dx: 0, dy: 500))
                    }
                case "s": // 蓝方攻击
                    blueCombo += 1
                    // MARK: 添加冷却检查
                    if !isBlueAttackOnCooldown { // 只有不在冷却中才能攻击
                         attack(from: bluePlayer, to: redPlayer, isBlue: true)
                    } else {
                         print("🟦 蓝方攻击冷却中...") // 冷却中按键提示
                    }


                case "i": // 红方跳跃
                    if isOnGround(redPlayer) && !isRedKnockedBack {
                        redPlayer.physicsBody?.applyImpulse(CGVector(dx: 0, dy: 500))
                    }
                case "k": // 红方攻击
                    redCombo += 1
                    // MARK: 添加冷却检查
                    if !isRedAttackOnCooldown { // 只有不在冷却中才能攻击
                         attack(from: redPlayer, to: bluePlayer, isBlue: false)
                    } else {
                         print("🟥 红方攻击冷却中...") // 冷却中按键提示
                    }

                default:
                    break
                }
            }
        }
    }

    // MARK: - keyUp(with event:)
    override func keyUp(with event: NSEvent) {
        guard let chars = event.characters else { return }
        for char in chars {
            keysPressed.remove(char)
        }
    }

    // MARK: - update(_ currentTime:)
    override func update(_ currentTime: TimeInterval) {
        let moveSpeed: CGFloat = 200.0

        if !isBlueKnockedBack {
            if keysPressed.contains("a") {
                bluePlayer.physicsBody?.velocity.dx = -moveSpeed
            } else if keysPressed.contains("d") {
                bluePlayer.physicsBody?.velocity.dx = moveSpeed
            } else {
                bluePlayer.physicsBody?.velocity.dx = 0
            }
        }

        if !isRedKnockedBack {
            if keysPressed.contains("j") {
                redPlayer.physicsBody?.velocity.dx = -moveSpeed
            } else if keysPressed.contains("l") {
                redPlayer.physicsBody?.velocity.dx = moveSpeed
            } else {
                redPlayer.physicsBody?.velocity.dx = 0
            }
        }
    }

    // MARK: - attack(from:to:isBlue:) - 处理攻击逻辑和冷却
    func attack(from attacker: SKSpriteNode, to target: SKSpriteNode, isBlue: Bool) {
        let hitRange: CGFloat = 80
        let attackerX = attacker.position.x
        let targetX = target.position.x

        // 如果攻击命中
        if abs(attackerX - targetX) < hitRange {
            var damage = 10
            let comboBonus = 15
            let minComboForBonus = 3

            let knockbackImpulse: CGFloat = 500
            let verticalImpulse: CGFloat = 250
            let direction = attacker.position.x < target.position.x ? 1 : -1

            // MARK: 处理伤害、连击、设置击退状态、设置冷却和定时器
            if isBlue { // 蓝方攻击红方
                if blueCombo >= minComboForBonus {
                    damage += comboBonus
                    blueCombo = 0
                    print("🟦 蓝方触发连击!")
                }
                redHP -= damage
                redHP = max(redHP, 0)
                hpLabelRed.text = "红方 HP: \(redHP)"
                print("🟦 蓝方出拳，造成 \(damage) 点伤害，红方剩余 HP: \(redHP)")

                // 应用击退到红方
                target.physicsBody?.applyImpulse(CGVector(dx: knockbackImpulse * CGFloat(direction), dy: verticalImpulse))
                // 设置红方被击退状态
                isRedKnockedBack = true
                let knockbackDuration: TimeInterval = 0.5
                let resetKnockbackAction = SKAction.sequence([
                    SKAction.wait(forDuration: knockbackDuration),
                    SKAction.run { self.isRedKnockedBack = false; print("🟥 红方击退状态结束") }
                ])
                target.run(resetKnockbackAction, withKey: "knockbackResetAction")

                // MARK: 设置蓝方攻击冷却
                self.isBlueAttackOnCooldown = true // 设置蓝方冷却标志
                let blueCooldownAction = SKAction.sequence([
                    SKAction.wait(forDuration: attackCooldownDuration), // 等待冷却时间
                    SKAction.run {
                        self.isBlueAttackOnCooldown = false // 重置冷却标志
                        print("🟦 蓝方攻击冷却结束")
                    }
                ])
                attacker.run(blueCooldownAction, withKey: "blueAttackCooldown") // 在攻击者节点上运行冷却计时动作


            } else { // 红方攻击蓝方
                if redCombo >= minComboForBonus {
                    damage += comboBonus
                    redCombo = 0
                    print("🟥 红方触发连击!")
                }
                blueHP -= damage
                blueHP = max(blueHP, 0)
                hpLabelBlue.text = "蓝方 HP: \(blueHP)"
                print("🟥 红方出拳，造成 \(damage) 点伤害，蓝方剩余 HP: \(blueHP)")

                 // 应用击退到蓝方
                target.physicsBody?.applyImpulse(CGVector(dx: knockbackImpulse * CGFloat(direction), dy: verticalImpulse))
                // 设置蓝方被击退状态
                isBlueKnockedBack = true
                let knockbackDuration: TimeInterval = 0.5
                 let resetKnockbackAction = SKAction.sequence([
                    SKAction.wait(forDuration: knockbackDuration),
                    SKAction.run { self.isBlueKnockedBack = false; print("🟦 蓝方击退状态结束") }
                ])
                target.run(resetKnockbackAction, withKey: "knockbackResetAction")

                // MARK: 设置红方攻击冷却
                self.isRedAttackOnCooldown = true // 设置红方冷却标志
                let redCooldownAction = SKAction.sequence([
                    SKAction.wait(forDuration: attackCooldownDuration), // 等待冷却时间
                    SKAction.run {
                        self.isRedAttackOnCooldown = false // 重置冷却标志
                        print("🟥 红方攻击冷却结束")
                    }
                ])
                attacker.run(redCooldownAction, withKey: "redAttackCooldown") // 在攻击者节点上运行冷却计时动作
            }

            // 以下代码在命中后都会执行
            showHitEffect(at: target.position) // 显示击中效果

            let flash = SKAction.sequence([ // 玩家闪烁效果
                SKAction.fadeAlpha(to: 0.3, duration: 0.05),
                SKAction.fadeAlpha(to: 1.0, duration: 0.05)
            ])
            target.run(flash)

            checkWin() // 检查胜利条件

        } // End if abs(attackerX - targetX) < hitRange

        // 注意：连击计数 (blueCombo, redCombo) 在命中判断之外，
        // 意味着即使攻击没有命中，连击数也会增加。这是你原代码的逻辑，保留。
    }

    // MARK: - showHitEffect(at position:)
    func showHitEffect(at position: CGPoint) {
        let effect = SKShapeNode(circleOfRadius: 20)
        effect.position = position
        effect.fillColor = .yellow
        effect.strokeColor = .clear
        effect.zPosition = 10
        addChild(effect)

        let expand = SKAction.scale(to: 2.0, duration: 0.1)
        let fade = SKAction.fadeOut(withDuration: 0.1)
        let remove = SKAction.removeFromParent()
        let sequence = SKAction.sequence([expand, fade, remove])
        effect.run(sequence)
    }

    // MARK: - checkWin()
    func checkWin() {
        if blueHP <= 0 {
            showVictoryMessage("🔴 红方胜利！")
        } else if redHP <= 0 {
            showVictoryMessage("🔵 蓝方胜利！")
        }
    }

    // MARK: - showVictoryMessage(_ text:)
    func showVictoryMessage(_ text: String) {
        if isPaused { return }
        isPaused = true

        let label = SKLabelNode(text: text)
        label.fontSize = 36
        label.fontColor = .black
        label.position = CGPoint(x: size.width / 2, y: size.height / 2)
        label.zPosition = 20
        addChild(label)
    }

    // MARK: - isOnGround(_ player:)
    func isOnGround(_ player: SKSpriteNode) -> Bool {
        guard let body = player.physicsBody else { return false }

        let groundLevel: CGFloat = 25 + 50.0/2
        let playerBottomY = player.position.y - player.size.height / 2

        let isNearGround = playerBottomY <= groundLevel + 5.0
        let isVerticalVelocityLow = body.velocity.dy <= 10.0

        return isNearGround && isVerticalVelocityLow
    }

    // MARK: - SKPhysicsContactDelegate 方法 (如果需要更精确的地面检测，可以在这里实现)
    /*
    func didBegin(_ contact: SKPhysicsContact) {
       // ...
    }
    */
}
