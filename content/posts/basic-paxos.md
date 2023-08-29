---
title: "Basic Paxos"
date: 2023-08-24T20:59:37+08:00
draft: false
tags: ["分布式", "共识算法"]
---

## 角色

- 提议者（Proposer）：收到客户端的请求，提出相关的提案，试图让接受者接受该提案，发生冲突时进行协调，推动算法运行
- 接受者（Acceptor）：接受或拒绝提议者的提案，若超过半数的接受者接受提案，则该提案被批准
- 学习者（Learner）：只学习被批准的提案，不参与决议过程

![basic-paxos](/basic-paxos.png)

## 过程

### Prepare 阶段

- 某个提议者发起提案，向所有接受者发送 Prepare 请求，请求中附带数字 n 作为提案 ID
- 接受者收到 Prepare 请求后，将会给予提议者两个承诺和一个应答

#### 承诺

- 承诺不会再接受提案 ID 小于或等于 n 的 Prepare 请求
- 承诺不会再接受提案 ID 小于 n 的 Accept 请求

#### 应答

- 提案 ID 大于之前接受的所有提案 ID，则返回 Promise 响应，附带之前接受的提案 ID 和对应的提案值
- 提案 ID 小于之前接受的所有提案 ID，则返回 Reject 响应

> 如果 Promise 响应中中的提案 ID 大于提议者发起的提案 ID，则提议者需要放弃当前提案，使用 Promise 响应中的提案 ID 和提案值发起 Accept 请求

### Accept 阶段

- 如果提议者收到了超过半数的接受者的 Promise 响应，则可以向这些接受者发送 Accept 请求
- 接受者收到 Accept 请求后，如果请求中的提案 ID 大于等于之前接受的所有提案 ID，则接受该提案，并将提案 ID 和提案值存储起来
- 如果接受者收到了超过半数的 Accept 请求，则该提案被批准

## Go 实现

https://github.com/ramzeng/basic-paxos

### Proposer

```go
package main

import "fmt"

// Proposer 提议者
type Proposer struct {
	// 节点 ID
	id int
	// 最大轮次
	round int
	// 提案编号
	number int
	// 接受者 ID 列表
	acceptors []int
}

func (p *Proposer) Propose(value interface{}) interface{} {
	p.round++
	p.number = p.proposalNumber()

	// Prepare 阶段
	preparedAcceptorsCount := 0
	maxProposerNumber := 0

	for _, acceptor := range p.acceptors {
		message := Message{
			ProposalNumber: p.number,
			From:           p.id,
			To:             acceptor,
		}

		reply := &Reply{}

		if err := call(
			fmt.Sprintf("127.0.0.1:%d", acceptor),
			"Acceptor.Prepare",
			message,
			reply,
		); err != nil {
			continue
		}

		if reply.OK {
			preparedAcceptorsCount++
			// 如果收到的提案编号比当前的大，就更新当前的提案编号和提案值
			if reply.ProposalNumber > maxProposerNumber {
				maxProposerNumber = reply.ProposalNumber
				value = reply.ProposalValue
			}
		}

		if preparedAcceptorsCount > p.halfAcceptorsCount() {
			break
		}
	}

	// Accept 阶段
	acceptedAcceptorsCount := 0

	if preparedAcceptorsCount > p.halfAcceptorsCount() {
		for _, acceptor := range p.acceptors {
			message := Message{
				ProposalNumber: p.number,
				ProposalValue:  value,
				From:           p.id,
				To:             acceptor,
			}

			reply := &Reply{}

			if err := call(
				fmt.Sprintf("127.0.0.1:%d", acceptor),
				"Acceptor.Accept",
				message,
				reply,
			); err != nil {
				continue
			}

			if reply.OK {
				acceptedAcceptorsCount++
			}
		}
	}

	if acceptedAcceptorsCount > p.halfAcceptorsCount() {
		return value
	}

	return nil
}

func (p *Proposer) proposalNumber() int {
	// 提案编号 = (轮次,节点 ID)
	return p.round<<16 | p.id
}

func (p *Proposer) halfAcceptorsCount() int {
	return len(p.acceptors) / 2
}
```

### Acceptor

```go
package main

import (
	"fmt"
	"net"
	"net/rpc"
)

// Acceptor 接受者
type Acceptor struct {
	listener net.Listener
	// 节点 ID
	id int
	// 承诺的提案编号，如果为 0，则表示没有收到过任何 Prepare 消息
	proposalProposal int
	// 已接受的提案编号，如果为 0，表示没有接受任何提案
	acceptedProposalNumber int
	// 已接受的提案值
	acceptedProposalValue interface{}
	// 学习者 ID 列表
	learners []int
}

func (a *Acceptor) Prepare(message *Message, reply *Reply) error {
	// 如果提案编号大于当前承诺的提案编号，则承诺提案编号
	if message.ProposalNumber > a.proposalProposal {
		a.proposalProposal = message.ProposalNumber

		// 返回已接受的提案编号和提案值
		reply.ProposalNumber = a.acceptedProposalNumber
		reply.ProposalValue = a.acceptedProposalValue
		reply.OK = true
	} else {
		// 否则，拒绝提案
		reply.OK = false
	}

	return nil
}

func (a *Acceptor) Accept(message *Message, reply *Reply) error {
	// 如果提案编号大于等于当前承诺的提案编号，则接受提案
	if message.ProposalNumber >= a.proposalProposal {
		a.proposalProposal = message.ProposalNumber

		// 记录已接受的提案编号和提案值
		a.acceptedProposalNumber = message.ProposalNumber
		a.acceptedProposalValue = message.ProposalValue

		reply.OK = true

		// 向所有学习者发送提案
		for _, learner := range a.learners {
			go func(learner int) {
				message.From = a.id
				message.To = learner

				if err := call(
					fmt.Sprintf("127.0.0.1:%d", learner),
					"Learner.Learn",
					message,
					&Reply{},
				); err != nil {
					return
				}
			}(learner)
		}
	} else {
		reply.OK = false
	}

	return nil
}

func (a *Acceptor) Serve() {
	server := rpc.NewServer()

	if err := server.Register(a); err != nil {
		panic(err)
	}

	if listen, err := net.Listen("tcp", fmt.Sprintf(":%d", a.id)); err != nil {
		panic(err)
	} else {
		a.listener = listen
	}

	go func() {
		for {
			connection, err := a.listener.Accept()

			if err != nil {
				continue
			}

			go server.ServeConn(connection)
		}
	}()
}

func (a *Acceptor) Close() {
	_ = a.listener.Close()
}

func NewAcceptor(id int, learners []int) *Acceptor {
	acceptor := &Acceptor{
		id:       id,
		learners: learners,
	}

	acceptor.Serve()

	return acceptor
}
```

### Learner

```go
package main

import (
	"fmt"
	"net"
	"net/rpc"
)

type Learner struct {
	listener net.Listener
	// 节点 ID
	id int
	// 已接受的提案
	acceptedMessages map[int]Message
}

func (l *Learner) Learn(message *Message, reply *Reply) error {
	m := l.acceptedMessages[message.From]

	// 如果提案的编号大于已接受的提案编号，则接受该提案
	if m.ProposalNumber < message.ProposalNumber {
		l.acceptedMessages[message.From] = *message
		reply.OK = true
	} else {
		reply.OK = false
	}

	return nil
}

func (l *Learner) Chosen() interface{} {
	acceptorsCount := make(map[int]int)
	acceptedMessages := make(map[int]Message)

	for _, message := range l.acceptedMessages {
		// 如果提案的编号不为 0，则接受该提案
		if message.ProposalNumber != 0 {
			acceptorsCount[message.ProposalNumber]++
			acceptedMessages[message.ProposalNumber] = message
		}
	}

	for proposalNumber, count := range acceptorsCount {
		if count > l.halfAcceptedMessagesCount() {
			return acceptedMessages[proposalNumber].ProposalValue
		}
	}

	return nil
}

func (l *Learner) Serve(id int) {
	server := rpc.NewServer()

	if err := server.Register(l); err != nil {
		panic(err)
	}

	if listen, err := net.Listen("tcp", fmt.Sprintf(":%d", id)); err != nil {
		panic(err)
	} else {
		l.listener = listen
	}

	go func() {
		for {
			connection, err := l.listener.Accept()

			if err != nil {
				continue
			}

			go server.ServeConn(connection)
		}
	}()
}

func (l *Learner) Close() {
	_ = l.listener.Close()
}

func (l *Learner) halfAcceptedMessagesCount() int {
	return len(l.acceptedMessages) / 2
}

func NewLearner(id int, acceptorIds []int) *Learner {
	learner := &Learner{
		id:               id,
		acceptedMessages: make(map[int]Message),
	}

	for _, acceptorId := range acceptorIds {
		learner.acceptedMessages[acceptorId] = Message{}
	}

	learner.Serve(id)

	return learner
}
```

## 参考

- http://icyfenix.cn/distribution/consensus/paxos.html
